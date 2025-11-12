# Questions 11-14: TypeScript Advanced Topics

## Question 11: How do you implement design patterns in TypeScript?

### Answer:

**Design Patterns** in TypeScript leverage the type system to create robust, reusable solutions. TypeScript's features like interfaces, generics, and decorators enhance traditional patterns.

### Key Patterns:
1. **Singleton** - Single instance
2. **Factory** - Object creation
3. **Observer** - Event subscription
4. **Repository** - Data access abstraction
5. **Builder** - Complex object construction

### Banking Scenario: Design Patterns at Emirates NBD

```typescript
console.log('=== Design Patterns in TypeScript - Emirates NBD ===\n');

// =============================================================================
// 1. SINGLETON PATTERN
// =============================================================================

console.log('1. SINGLETON PATTERN:\n');

class DatabaseConnection {
    private static instance: DatabaseConnection;
    private connectionId: string;
    private isConnected: boolean = false;
    
    private constructor() {
        this.connectionId = `CONN-${Date.now()}`;
        console.log(`  Database connection created: ${this.connectionId}`);
    }
    
    public static getInstance(): DatabaseConnection {
        if (!DatabaseConnection.instance) {
            DatabaseConnection.instance = new DatabaseConnection();
        }
        return DatabaseConnection.instance;
    }
    
    public connect(): void {
        if (!this.isConnected) {
            this.isConnected = true;
            console.log(`  Connected to database (${this.connectionId})`);
        }
    }
    
    public query(sql: string): any[] {
        if (!this.isConnected) {
            throw new Error('Not connected to database');
        }
        console.log(`  Executing: ${sql}`);
        return [];
    }
}

// Usage
const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();

console.log(`Same instance: ${db1 === db2}`);
db1.connect();
db2.query('SELECT * FROM accounts');
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. FACTORY PATTERN
// =============================================================================

console.log('2. FACTORY PATTERN:\n');

// Product interface
interface BankAccount {
    accountNumber: string;
    balance: number;
    getAccountType(): string;
    calculateInterest(): number;
    getDetails(): string;
}

// Concrete products
class SavingsAccount implements BankAccount {
    constructor(
        public accountNumber: string,
        public balance: number,
        private interestRate: number = 2.5
    ) {}
    
    getAccountType(): string {
        return 'SAVINGS';
    }
    
    calculateInterest(): number {
        return this.balance * (this.interestRate / 100);
    }
    
    getDetails(): string {
        return `Savings Account ${this.accountNumber}: ${this.balance} AED (${this.interestRate}% interest)`;
    }
}

class CurrentAccount implements BankAccount {
    constructor(
        public accountNumber: string,
        public balance: number,
        private overdraftLimit: number = 10000
    ) {}
    
    getAccountType(): string {
        return 'CURRENT';
    }
    
    calculateInterest(): number {
        return 0; // No interest on current accounts
    }
    
    getDetails(): string {
        return `Current Account ${this.accountNumber}: ${this.balance} AED (Overdraft: ${this.overdraftLimit} AED)`;
    }
}

class FixedDepositAccount implements BankAccount {
    constructor(
        public accountNumber: string,
        public balance: number,
        private interestRate: number = 5.0,
        private maturityMonths: number = 12
    ) {}
    
    getAccountType(): string {
        return 'FIXED_DEPOSIT';
    }
    
    calculateInterest(): number {
        return this.balance * (this.interestRate / 100) * (this.maturityMonths / 12);
    }
    
    getDetails(): string {
        return `Fixed Deposit ${this.accountNumber}: ${this.balance} AED (${this.interestRate}% for ${this.maturityMonths} months)`;
    }
}

// Factory
class AccountFactory {
    static createAccount(
        type: 'SAVINGS' | 'CURRENT' | 'FIXED',
        accountNumber: string,
        balance: number,
        options?: any
    ): BankAccount {
        switch (type) {
            case 'SAVINGS':
                return new SavingsAccount(accountNumber, balance, options?.interestRate);
            case 'CURRENT':
                return new CurrentAccount(accountNumber, balance, options?.overdraftLimit);
            case 'FIXED':
                return new FixedDepositAccount(
                    accountNumber,
                    balance,
                    options?.interestRate,
                    options?.maturityMonths
                );
            default:
                throw new Error(`Unknown account type: ${type}`);
        }
    }
}

// Usage
console.log('Creating accounts using Factory:\n');
const savings = AccountFactory.createAccount('SAVINGS', 'SAV-001', 50000);
const current = AccountFactory.createAccount('CURRENT', 'CUR-001', 30000);
const fixed = AccountFactory.createAccount('FIXED', 'FD-001', 100000, {
    interestRate: 5.5,
    maturityMonths: 24
});

console.log(`  ${savings.getDetails()}`);
console.log(`    Interest: ${savings.calculateInterest()} AED\n`);
console.log(`  ${current.getDetails()}`);
console.log(`    Interest: ${current.calculateInterest()} AED\n`);
console.log(`  ${fixed.getDetails()}`);
console.log(`    Interest: ${fixed.calculateInterest()} AED\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. OBSERVER PATTERN
// =============================================================================

console.log('3. OBSERVER PATTERN:\n');

// Observer interface
interface Observer<T> {
    update(data: T): void;
}

// Subject interface
interface Subject<T> {
    attach(observer: Observer<T>): void;
    detach(observer: Observer<T>): void;
    notify(data: T): void;
}

// Event types
interface TransactionEvent {
    type: 'DEPOSIT' | 'WITHDRAWAL' | 'TRANSFER';
    accountNumber: string;
    amount: number;
    timestamp: Date;
}

// Concrete subject
class TransactionMonitor implements Subject<TransactionEvent> {
    private observers: Observer<TransactionEvent>[] = [];
    
    attach(observer: Observer<TransactionEvent>): void {
        this.observers.push(observer);
        console.log(`  Observer attached (Total: ${this.observers.length})`);
    }
    
    detach(observer: Observer<TransactionEvent>): void {
        const index = this.observers.indexOf(observer);
        if (index > -1) {
            this.observers.splice(index, 1);
            console.log(`  Observer detached (Total: ${this.observers.length})`);
        }
    }
    
    notify(data: TransactionEvent): void {
        console.log(`  Notifying ${this.observers.length} observers...`);
        this.observers.forEach(observer => observer.update(data));
    }
    
    recordTransaction(event: TransactionEvent): void {
        console.log(`\n  Transaction: ${event.type} - ${event.amount} AED on ${event.accountNumber}`);
        this.notify(event);
    }
}

// Concrete observers
class EmailNotificationService implements Observer<TransactionEvent> {
    update(data: TransactionEvent): void {
        console.log(`    📧 Email sent: ${data.type} of ${data.amount} AED`);
    }
}

class SMSNotificationService implements Observer<TransactionEvent> {
    update(data: TransactionEvent): void {
        console.log(`    📱 SMS sent: ${data.type} transaction alert`);
    }
}

class AuditLogger implements Observer<TransactionEvent> {
    update(data: TransactionEvent): void {
        console.log(`    📝 Audit log: [${data.timestamp.toISOString()}] ${data.type} - ${data.amount} AED`);
    }
}

class FraudDetectionService implements Observer<TransactionEvent> {
    update(data: TransactionEvent): void {
        if (data.amount > 50000) {
            console.log(`    🚨 Fraud alert: Large transaction detected (${data.amount} AED)`);
        }
    }
}

// Usage
console.log('Setting up observers:\n');
const monitor = new TransactionMonitor();
const emailService = new EmailNotificationService();
const smsService = new SMSNotificationService();
const auditLogger = new AuditLogger();
const fraudDetection = new FraudDetectionService();

monitor.attach(emailService);
monitor.attach(smsService);
monitor.attach(auditLogger);
monitor.attach(fraudDetection);

monitor.recordTransaction({
    type: 'WITHDRAWAL',
    accountNumber: 'ACC-123456',
    amount: 75000,
    timestamp: new Date()
});

console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. REPOSITORY PATTERN
// =============================================================================

console.log('4. REPOSITORY PATTERN:\n');

// Generic repository interface
interface IRepository<T, ID> {
    findById(id: ID): Promise<T | null>;
    findAll(): Promise<T[]>;
    save(entity: T): Promise<T>;
    update(id: ID, entity: Partial<T>): Promise<T>;
    delete(id: ID): Promise<void>;
}

// Entity
interface Account {
    id: string;
    accountNumber: string;
    customerId: string;
    balance: number;
    type: string;
    createdAt: Date;
}

// Concrete repository
class AccountRepository implements IRepository<Account, string> {
    private accounts: Map<string, Account> = new Map();
    
    async findById(id: string): Promise<Account | null> {
        console.log(`  Finding account by ID: ${id}`);
        return this.accounts.get(id) || null;
    }
    
    async findAll(): Promise<Account[]> {
        console.log(`  Finding all accounts`);
        return Array.from(this.accounts.values());
    }
    
    async save(entity: Account): Promise<Account> {
        console.log(`  Saving account: ${entity.accountNumber}`);
        this.accounts.set(entity.id, entity);
        return entity;
    }
    
    async update(id: string, updates: Partial<Account>): Promise<Account> {
        console.log(`  Updating account: ${id}`);
        const account = await this.findById(id);
        if (!account) {
            throw new Error('Account not found');
        }
        const updated = { ...account, ...updates };
        this.accounts.set(id, updated);
        return updated;
    }
    
    async delete(id: string): Promise<void> {
        console.log(`  Deleting account: ${id}`);
        this.accounts.delete(id);
    }
    
    // Custom query methods
    async findByCustomerId(customerId: string): Promise<Account[]> {
        console.log(`  Finding accounts for customer: ${customerId}`);
        return Array.from(this.accounts.values())
            .filter(acc => acc.customerId === customerId);
    }
    
    async findByType(type: string): Promise<Account[]> {
        console.log(`  Finding ${type} accounts`);
        return Array.from(this.accounts.values())
            .filter(acc => acc.type === type);
    }
}

// Usage
(async () => {
    const accountRepo = new AccountRepository();
    
    console.log('Repository operations:\n');
    
    const newAccount: Account = {
        id: 'ACC-001',
        accountNumber: 'ACC-123456789',
        customerId: 'CUST-001',
        balance: 50000,
        type: 'SAVINGS',
        createdAt: new Date()
    };
    
    await accountRepo.save(newAccount);
    const found = await accountRepo.findById('ACC-001');
    console.log(`  Found: ${found?.accountNumber}\n`);
    
    await accountRepo.update('ACC-001', { balance: 55000 });
    console.log('  Balance updated\n');
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 5. BUILDER PATTERN
    // =============================================================================
    
    console.log('5. BUILDER PATTERN:\n');
    
    // Complex object
    interface TransferRequest {
        fromAccountId: string;
        toAccountId: string;
        amount: number;
        currency: string;
        description?: string;
        scheduled?: Date;
        recurring?: {
            frequency: 'DAILY' | 'WEEKLY' | 'MONTHLY';
            endDate: Date;
        };
        notifications?: {
            email: boolean;
            sms: boolean;
            push: boolean;
        };
        priority?: 'LOW' | 'NORMAL' | 'HIGH';
    }
    
    // Builder
    class TransferRequestBuilder {
        private request: Partial<TransferRequest> = {
            currency: 'AED',
            priority: 'NORMAL'
        };
        
        from(accountId: string): this {
            this.request.fromAccountId = accountId;
            return this;
        }
        
        to(accountId: string): this {
            this.request.toAccountId = accountId;
            return this;
        }
        
        amount(value: number): this {
            this.request.amount = value;
            return this;
        }
        
        currency(curr: string): this {
            this.request.currency = curr;
            return this;
        }
        
        description(desc: string): this {
            this.request.description = desc;
            return this;
        }
        
        scheduleFor(date: Date): this {
            this.request.scheduled = date;
            return this;
        }
        
        makeRecurring(frequency: 'DAILY' | 'WEEKLY' | 'MONTHLY', endDate: Date): this {
            this.request.recurring = { frequency, endDate };
            return this;
        }
        
        withNotifications(email: boolean, sms: boolean, push: boolean): this {
            this.request.notifications = { email, sms, push };
            return this;
        }
        
        priority(level: 'LOW' | 'NORMAL' | 'HIGH'): this {
            this.request.priority = level;
            return this;
        }
        
        build(): TransferRequest {
            if (!this.request.fromAccountId || !this.request.toAccountId || !this.request.amount) {
                throw new Error('Missing required fields');
            }
            return this.request as TransferRequest;
        }
    }
    
    // Usage
    console.log('Building complex transfer request:\n');
    
    const transfer = new TransferRequestBuilder()
        .from('ACC-123456')
        .to('ACC-789012')
        .amount(5000)
        .currency('AED')
        .description('Monthly rent payment')
        .makeRecurring('MONTHLY', new Date('2026-12-31'))
        .withNotifications(true, true, false)
        .priority('HIGH')
        .build();
    
    console.log('  Transfer Request:');
    console.log(`    From: ${transfer.fromAccountId}`);
    console.log(`    To: ${transfer.toAccountId}`);
    console.log(`    Amount: ${transfer.amount} ${transfer.currency}`);
    console.log(`    Description: ${transfer.description}`);
    console.log(`    Recurring: ${transfer.recurring?.frequency}`);
    console.log(`    Priority: ${transfer.priority}`);
    console.log(`    Notifications: Email=${transfer.notifications?.email}, SMS=${transfer.notifications?.sms}\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // DESIGN PATTERNS SUMMARY
    // =============================================================================
    
    console.log('DESIGN PATTERNS SUMMARY:\n');
    
    console.log('Singleton Pattern:');
    console.log('  • Single instance guarantee');
    console.log('  • Private constructor');
    console.log('  • Static getInstance() method');
    console.log('  • Use for: Database connections, config\n');
    
    console.log('Factory Pattern:');
    console.log('  • Centralized object creation');
    console.log('  • Encapsulates instantiation logic');
    console.log('  • Easy to extend with new types');
    console.log('  • Use for: Multiple related classes\n');
    
    console.log('Observer Pattern:');
    console.log('  • Publish-subscribe mechanism');
    console.log('  • Loose coupling');
    console.log('  • Multiple observers');
    console.log('  • Use for: Event systems, notifications\n');
    
    console.log('Repository Pattern:');
    console.log('  • Data access abstraction');
    console.log('  • Generic CRUD operations');
    console.log('  • Testable and mockable');
    console.log('  • Use for: Database operations\n');
    
    console.log('Builder Pattern:');
    console.log('  • Step-by-step object construction');
    console.log('  • Fluent interface');
    console.log('  • Optional parameters');
    console.log('  • Use for: Complex objects\n');
    
    console.log('TypeScript Benefits:');
    console.log('  ✓ Interfaces enforce contracts');
    console.log('  ✓ Generics for reusability');
    console.log('  ✓ Type safety throughout');
    console.log('  ✓ Better IDE support');
    console.log('  ✓ Compile-time checks');
    
    console.log();
})();
```

---

## Question 12: How do you handle asynchronous operations and Promises in TypeScript?

### Answer:

**Asynchronous TypeScript** provides type-safe handling of Promises, async/await, and concurrent operations. Proper typing prevents runtime errors and improves code reliability.

### Key Concepts:
1. **Promise Types** - Promise<T> typing
2. **Async/Await** - Asynchronous functions
3. **Error Handling** - try/catch with types
4. **Parallel Operations** - Promise.all, Promise.race
5. **Custom Async Patterns** - Retry, timeout, queue

### Banking Scenario: Async Operations at Emirates NBD

```typescript
console.log('=== Asynchronous TypeScript - Emirates NBD ===\n');

// =============================================================================
// 1. PROMISE TYPING
// =============================================================================

console.log('1. PROMISE TYPING:\n');

// Basic Promise types
function getAccountBalance(accountId: string): Promise<number> {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (accountId.startsWith('ACC-')) {
                resolve(50000);
            } else {
                reject(new Error('Invalid account ID'));
            }
        }, 100);
    });
}

// Promise with complex types
interface AccountDetails {
    id: string;
    balance: number;
    type: string;
    lastTransaction: Date;
}

async function getAccountDetails(accountId: string): Promise<AccountDetails> {
    await new Promise(resolve => setTimeout(resolve, 100));
    
    return {
        id: accountId,
        balance: 50000,
        type: 'SAVINGS',
        lastTransaction: new Date()
    };
}

// Promise returning functions
console.log('Promise Types:\n');
console.log('  Promise<number>        - Returns number');
console.log('  Promise<string>        - Returns string');
console.log('  Promise<Account>       - Returns Account object');
console.log('  Promise<void>          - Returns nothing');
console.log('  Promise<Account | null> - Returns Account or null\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. ASYNC/AWAIT
// =============================================================================

console.log('2. ASYNC/AWAIT:\n');

// Async function with typed return
async function transferFunds(
    fromAccount: string,
    toAccount: string,
    amount: number
): Promise<{ transactionId: string; status: string }> {
    console.log(`  Transferring ${amount} AED from ${fromAccount} to ${toAccount}`);
    
    // Simulate async operations
    await new Promise(resolve => setTimeout(resolve, 100));
    
    return {
        transactionId: `TXN-${Date.now()}`,
        status: 'COMPLETED'
    };
}

// Sequential operations
async function processSequential() {
    console.log('\nSequential operations:');
    
    const balance1 = await getAccountBalance('ACC-001');
    console.log(`  Account 1 balance: ${balance1} AED`);
    
    const balance2 = await getAccountBalance('ACC-002');
    console.log(`  Account 2 balance: ${balance2} AED`);
    
    const result = await transferFunds('ACC-001', 'ACC-002', 1000);
    console.log(`  Transfer ${result.status}: ${result.transactionId}\n`);
}

// Parallel operations
async function processParallel() {
    console.log('Parallel operations:');
    
    const [balance1, balance2, details] = await Promise.all([
        getAccountBalance('ACC-001'),
        getAccountBalance('ACC-002'),
        getAccountDetails('ACC-001')
    ]);
    
    console.log(`  Account 1: ${balance1} AED`);
    console.log(`  Account 2: ${balance2} AED`);
    console.log(`  Details: ${details.type} account\n`);
}

(async () => {
    await processSequential();
    await processParallel();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 3. ERROR HANDLING
    // =============================================================================
    
    console.log('3. ERROR HANDLING:\n');
    
    // Custom error types
    class TransactionError extends Error {
        constructor(
            message: string,
            public code: string,
            public accountId?: string
        ) {
            super(message);
            this.name = 'TransactionError';
        }
    }
    
    class InsufficientFundsError extends TransactionError {
        constructor(accountId: string, required: number, available: number) {
            super(
                `Insufficient funds: Required ${required} AED, Available ${available} AED`,
                'INSUFFICIENT_FUNDS',
                accountId
            );
            this.name = 'InsufficientFundsError';
        }
    }
    
    // Function that may throw
    async function withdrawFunds(accountId: string, amount: number): Promise<number> {
        const balance = await getAccountBalance(accountId);
        
        if (balance < amount) {
            throw new InsufficientFundsError(accountId, amount, balance);
        }
        
        return balance - amount;
    }
    
    // Error handling with try/catch
    async function handleWithdrawal(accountId: string, amount: number) {
        try {
            console.log(`  Attempting withdrawal: ${amount} AED`);
            const newBalance = await withdrawFunds(accountId, amount);
            console.log(`  Success! New balance: ${newBalance} AED`);
        } catch (error) {
            if (error instanceof InsufficientFundsError) {
                console.log(`  ❌ ${error.message}`);
            } else if (error instanceof TransactionError) {
                console.log(`  ❌ Transaction error [${error.code}]: ${error.message}`);
            } else if (error instanceof Error) {
                console.log(`  ❌ Unexpected error: ${error.message}`);
            } else {
                console.log(`  ❌ Unknown error occurred`);
            }
        }
    }
    
    await handleWithdrawal('ACC-001', 30000);  // Success
    await handleWithdrawal('ACC-001', 100000); // Insufficient funds
    console.log();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 4. PROMISE COMBINATORS
    // =============================================================================
    
    console.log('4. PROMISE COMBINATORS:\n');
    
    // Promise.all - All must succeed
    console.log('Promise.all (all must succeed):');
    async function getAllBalances(accountIds: string[]): Promise<number[]> {
        return Promise.all(accountIds.map(id => getAccountBalance(id)));
    }
    
    const balances = await getAllBalances(['ACC-001', 'ACC-002', 'ACC-003']);
    console.log(`  All balances: ${balances.join(', ')} AED\n`);
    
    // Promise.allSettled - All results regardless of success/failure
    console.log('Promise.allSettled (all results):');
    async function checkAllAccounts(accountIds: string[]): Promise<void> {
        const results = await Promise.allSettled(
            accountIds.map(id => getAccountBalance(id))
        );
        
        results.forEach((result, index) => {
            if (result.status === 'fulfilled') {
                console.log(`  ${accountIds[index]}: ${result.value} AED`);
            } else {
                console.log(`  ${accountIds[index]}: Error - ${result.reason.message}`);
            }
        });
    }
    
    await checkAllAccounts(['ACC-001', 'INVALID', 'ACC-003']);
    console.log();
    
    // Promise.race - First to complete
    console.log('Promise.race (first to complete):');
    async function getQuickestResponse<T>(promises: Promise<T>[]): Promise<T> {
        return Promise.race(promises);
    }
    
    const fastest = await getQuickestResponse([
        getAccountBalance('ACC-001'),
        getAccountBalance('ACC-002')
    ]);
    console.log(`  Fastest response: ${fastest} AED\n`);
    
    // Promise.any - First to succeed
    console.log('Promise.any (first to succeed):');
    async function getFirstSuccess<T>(promises: Promise<T>[]): Promise<T> {
        return Promise.any(promises);
    }
    
    const firstSuccess = await getFirstSuccess([
        getAccountBalance('ACC-001'),
        getAccountBalance('INVALID'),
        getAccountBalance('ACC-003')
    ]);
    console.log(`  First success: ${firstSuccess} AED\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 5. ADVANCED ASYNC PATTERNS
    // =============================================================================
    
    console.log('5. ADVANCED PATTERNS:\n');
    
    // Retry pattern
    async function withRetry<T>(
        fn: () => Promise<T>,
        maxRetries: number = 3,
        delay: number = 1000
    ): Promise<T> {
        let lastError: Error | undefined;
        
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                return await fn();
            } catch (error) {
                lastError = error instanceof Error ? error : new Error('Unknown error');
                console.log(`  Attempt ${attempt} failed: ${lastError.message}`);
                
                if (attempt < maxRetries) {
                    await new Promise(resolve => setTimeout(resolve, delay));
                }
            }
        }
        
        throw lastError;
    }
    
    console.log('Retry Pattern:');
    let attemptCount = 0;
    try {
        await withRetry(async () => {
            attemptCount++;
            if (attemptCount < 2) {
                throw new Error('Temporary failure');
            }
            return 'Success';
        }, 3, 100);
        console.log(`  Operation succeeded after ${attemptCount} attempts\n`);
    } catch (error) {
        console.log(`  All retries failed\n`);
    }
    
    // Timeout pattern
    async function withTimeout<T>(
        promise: Promise<T>,
        timeoutMs: number
    ): Promise<T> {
        const timeout = new Promise<never>((_, reject) => {
            setTimeout(() => reject(new Error('Operation timed out')), timeoutMs);
        });
        
        return Promise.race([promise, timeout]);
    }
    
    console.log('Timeout Pattern:');
    try {
        await withTimeout(
            new Promise(resolve => setTimeout(resolve, 5000)),
            100
        );
    } catch (error) {
        console.log(`  ⏱️ ${error instanceof Error ? error.message : 'Timeout'}\n`);
    }
    
    // Queue pattern
    class AsyncQueue<T> {
        private queue: Array<() => Promise<T>> = [];
        private processing = false;
        
        add(task: () => Promise<T>): Promise<T> {
            return new Promise((resolve, reject) => {
                this.queue.push(async () => {
                    try {
                        const result = await task();
                        resolve(result);
                        return result;
                    } catch (error) {
                        reject(error);
                        throw error;
                    }
                });
                
                this.processQueue();
            });
        }
        
        private async processQueue(): Promise<void> {
            if (this.processing || this.queue.length === 0) {
                return;
            }
            
            this.processing = true;
            
            while (this.queue.length > 0) {
                const task = this.queue.shift();
                if (task) {
                    try {
                        await task();
                    } catch (error) {
                        // Error already handled in add()
                    }
                }
            }
            
            this.processing = false;
        }
    }
    
    console.log('Queue Pattern:');
    const queue = new AsyncQueue<string>();
    
    queue.add(async () => {
        await new Promise(resolve => setTimeout(resolve, 50));
        console.log('  Task 1 completed');
        return 'Task 1';
    });
    
    queue.add(async () => {
        await new Promise(resolve => setTimeout(resolve, 50));
        console.log('  Task 2 completed');
        return 'Task 2';
    });
    
    queue.add(async () => {
        await new Promise(resolve => setTimeout(resolve, 50));
        console.log('  Task 3 completed');
        return 'Task 3';
    });
    
    // Wait for all tasks to complete
    await new Promise(resolve => setTimeout(resolve, 200));
    console.log();
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // ASYNC SUMMARY
    // =============================================================================
    
    console.log('ASYNC TYPESCRIPT SUMMARY:\n');
    
    console.log('Promise Types:');
    console.log('  • Promise<T> - Generic promise type');
    console.log('  • async functions return Promise<T>');
    console.log('  • await extracts T from Promise<T>');
    console.log('  • Type safety throughout async chain\n');
    
    console.log('Error Handling:');
    console.log('  • Custom error classes');
    console.log('  • Type guards for error types');
    console.log('  • try/catch with typed errors');
    console.log('  • Proper error propagation\n');
    
    console.log('Combinators:');
    console.log('  • Promise.all - All must succeed');
    console.log('  • Promise.allSettled - All results');
    console.log('  • Promise.race - First to complete');
    console.log('  • Promise.any - First to succeed\n');
    
    console.log('Advanced Patterns:');
    console.log('  • Retry with exponential backoff');
    console.log('  • Timeout wrapper');
    console.log('  • Task queue');
    console.log('  • Debounce/throttle');
    console.log('  • Circuit breaker\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Always type Promise returns');
    console.log('  ✓ Handle errors explicitly');
    console.log('  ✓ Use Promise.all for parallel operations');
    console.log('  ✓ Implement timeout patterns');
    console.log('  ✓ Consider retry logic');
    console.log('  ✓ Avoid mixing callbacks and promises');
    console.log('  ✓ Use async/await over .then()');
    
    console.log();
})();
```

---

## Question 13: How do you use TypeScript with testing frameworks (Jest, Mocha)?

### Answer:

**TypeScript Testing** combines type safety with test frameworks. Proper typing of tests, mocks, and assertions improves test quality and maintainability.

### Key Areas:
1. **Jest Configuration** - TypeScript setup
2. **Test Typing** - Typed test suites
3. **Mocking** - Type-safe mocks
4. **Assertions** - Typed expectations
5. **Test Utilities** - Custom matchers

### Banking Scenario: Testing at Emirates NBD

```typescript
console.log('=== TypeScript Testing - Emirates NBD ===\n');

// =============================================================================
// 1. JEST CONFIGURATION
// =============================================================================

console.log('1. JEST CONFIGURATION:\n');

console.log('jest.config.js:\n');
console.log(`module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  transform: {
    '^.+\\\\.ts$': 'ts-jest'
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.interface.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  moduleNameMapper: {
    '^@models/(.*)$': '<rootDir>/src/models/$1',
    '^@services/(.*)$': '<rootDir>/src/services/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1'
  }
};\n`);

console.log('tsconfig.json for tests:\n');
console.log(`{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "types": ["jest", "node"]
  },
  "include": ["src/**/*", "tests/**/*"]
}\n`);

console.log('Dependencies:');
console.log('  npm install -D jest ts-jest @types/jest\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPED TEST SUITES
// =============================================================================

console.log('2. TYPED TEST SUITES:\n');

// Types for testing
interface Account {
    id: string;
    accountNumber: string;
    balance: number;
    type: 'SAVINGS' | 'CURRENT';
}

class AccountService {
    private accounts: Map<string, Account> = new Map();
    
    async createAccount(data: Omit<Account, 'id'>): Promise<Account> {
        const account: Account = {
            id: `ACC-${Date.now()}`,
            ...data
        };
        this.accounts.set(account.id, account);
        return account;
    }
    
    async getAccount(id: string): Promise<Account | null> {
        return this.accounts.get(id) || null;
    }
    
    async deposit(id: string, amount: number): Promise<Account> {
        const account = await this.getAccount(id);
        if (!account) {
            throw new Error('Account not found');
        }
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        
        account.balance += amount;
        return account;
    }
    
    async withdraw(id: string, amount: number): Promise<Account> {
        const account = await this.getAccount(id);
        if (!account) {
            throw new Error('Account not found');
        }
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        if (account.balance < amount) {
            throw new Error('Insufficient funds');
        }
        
        account.balance -= amount;
        return account;
    }
}

// Test suite
console.log('Test Suite Example:\n');
console.log(`
describe('AccountService', () => {
  let service: AccountService;
  let testAccount: Account;
  
  beforeEach(async () => {
    service = new AccountService();
    testAccount = await service.createAccount({
      accountNumber: 'ACC-123',
      balance: 10000,
      type: 'SAVINGS'
    });
  });
  
  describe('createAccount', () => {
    it('should create account with correct data', async () => {
      const account = await service.createAccount({
        accountNumber: 'ACC-456',
        balance: 5000,
        type: 'CURRENT'
      });
      
      expect(account).toMatchObject({
        accountNumber: 'ACC-456',
        balance: 5000,
        type: 'CURRENT'
      });
      expect(account.id).toBeDefined();
    });
  });
  
  describe('deposit', () => {
    it('should increase balance', async () => {
      const updated = await service.deposit(testAccount.id, 5000);
      expect(updated.balance).toBe(15000);
    });
    
    it('should throw error for negative amount', async () => {
      await expect(service.deposit(testAccount.id, -1000))
        .rejects.toThrow('Amount must be positive');
    });
  });
  
  describe('withdraw', () => {
    it('should decrease balance', async () => {
      const updated = await service.withdraw(testAccount.id, 3000);
      expect(updated.balance).toBe(7000);
    });
    
    it('should throw error for insufficient funds', async () => {
      await expect(service.withdraw(testAccount.id, 20000))
        .rejects.toThrow('Insufficient funds');
    });
  });
});\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. MOCKING
// =============================================================================

console.log('3. TYPE-SAFE MOCKING:\n');

// Mock types
interface TransactionRepository {
    save(transaction: Transaction): Promise<void>;
    findById(id: string): Promise<Transaction | null>;
}

interface Transaction {
    id: string;
    accountId: string;
    amount: number;
    type: 'DEPOSIT' | 'WITHDRAWAL';
    timestamp: Date;
}

class TransactionService {
    constructor(private repository: TransactionRepository) {}
    
    async recordTransaction(
        accountId: string,
        amount: number,
        type: 'DEPOSIT' | 'WITHDRAWAL'
    ): Promise<Transaction> {
        const transaction: Transaction = {
            id: `TXN-${Date.now()}`,
            accountId,
            amount,
            type,
            timestamp: new Date()
        };
        
        await this.repository.save(transaction);
        return transaction;
    }
}

console.log('Mock Implementation:\n');
console.log(`
// Create typed mock
const mockRepository: jest.Mocked<TransactionRepository> = {
  save: jest.fn(),
  findById: jest.fn()
};

describe('TransactionService', () => {
  let service: TransactionService;
  
  beforeEach(() => {
    service = new TransactionService(mockRepository);
    jest.clearAllMocks();
  });
  
  it('should save transaction', async () => {
    mockRepository.save.mockResolvedValue(undefined);
    
    const transaction = await service.recordTransaction(
      'ACC-123',
      5000,
      'DEPOSIT'
    );
    
    expect(mockRepository.save).toHaveBeenCalledWith(
      expect.objectContaining({
        accountId: 'ACC-123',
        amount: 5000,
        type: 'DEPOSIT'
      })
    );
    expect(transaction).toBeDefined();
  });
  
  it('should handle repository errors', async () => {
    mockRepository.save.mockRejectedValue(
      new Error('Database error')
    );
    
    await expect(
      service.recordTransaction('ACC-123', 5000, 'DEPOSIT')
    ).rejects.toThrow('Database error');
  });
});\n`);

console.log('Partial Mocks:\n');
console.log(`
const partialMock: Partial<jest.Mocked<AccountService>> = {
  getAccount: jest.fn(),
  deposit: jest.fn()
};

partialMock.getAccount?.mockResolvedValue({
  id: 'ACC-123',
  accountNumber: 'ACC-123456',
  balance: 10000,
  type: 'SAVINGS'
});\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. CUSTOM MATCHERS
// =============================================================================

console.log('4. CUSTOM MATCHERS:\n');

console.log('Type-safe custom matcher:\n');
console.log(`
declare global {
  namespace jest {
    interface Matchers<R> {
      toHaveSufficientBalance(amount: number): R;
      toBeValidAccount(): R;
    }
  }
}

expect.extend({
  toHaveSufficientBalance(
    received: Account,
    amount: number
  ) {
    const pass = received.balance >= amount;
    
    return {
      pass,
      message: () =>
        pass
          ? \`Expected \${received.accountNumber} not to have sufficient balance\`
          : \`Expected \${received.accountNumber} to have at least \${amount} AED, but has \${received.balance} AED\`
    };
  },
  
  toBeValidAccount(received: any) {
    const isValid =
      typeof received === 'object' &&
      typeof received.id === 'string' &&
      typeof received.accountNumber === 'string' &&
      typeof received.balance === 'number' &&
      ['SAVINGS', 'CURRENT'].includes(received.type);
    
    return {
      pass: isValid,
      message: () =>
        isValid
          ? 'Expected not to be valid account'
          : 'Expected to be valid account structure'
    };
  }
});

// Usage
it('should have sufficient balance', () => {
  expect(account).toHaveSufficientBalance(5000);
});

it('should be valid account', () => {
  expect(account).toBeValidAccount();
});\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. TEST UTILITIES
// =============================================================================

console.log('5. TEST UTILITIES:\n');

console.log('Test Helpers:\n');
console.log(`
// Factory functions
function createTestAccount(
  overrides?: Partial<Account>
): Account {
  return {
    id: 'TEST-001',
    accountNumber: 'ACC-123456',
    balance: 10000,
    type: 'SAVINGS',
    ...overrides
  };
}

function createTestTransaction(
  overrides?: Partial<Transaction>
): Transaction {
  return {
    id: 'TXN-001',
    accountId: 'ACC-123',
    amount: 5000,
    type: 'DEPOSIT',
    timestamp: new Date(),
    ...overrides
  };
}

// Builder pattern for tests
class AccountBuilder {
  private account: Partial<Account> = {
    balance: 10000,
    type: 'SAVINGS'
  };
  
  withId(id: string): this {
    this.account.id = id;
    return this;
  }
  
  withBalance(balance: number): this {
    this.account.balance = balance;
    return this;
  }
  
  withType(type: 'SAVINGS' | 'CURRENT'): this {
    this.account.type = type;
    return this;
  }
  
  build(): Account {
    return this.account as Account;
  }
}

// Usage
const account = new AccountBuilder()
  .withId('ACC-123')
  .withBalance(50000)
  .withType('CURRENT')
  .build();\n`);

console.log('Async Test Utilities:\n');
console.log(`
// Wait for condition
async function waitFor<T>(
  fn: () => T | Promise<T>,
  options: {
    timeout?: number;
    interval?: number;
  } = {}
): Promise<T> {
  const { timeout = 5000, interval = 50 } = options;
  const startTime = Date.now();
  
  while (Date.now() - startTime < timeout) {
    try {
      const result = await fn();
      if (result) return result;
    } catch (error) {
      // Continue trying
    }
    await new Promise(resolve => setTimeout(resolve, interval));
  }
  
  throw new Error(\`Timeout waiting for condition\`);
}

// Usage in tests
it('should eventually update balance', async () => {
  const account = await waitFor(
    () => service.getAccount('ACC-123'),
    { timeout: 3000 }
  );
  
  expect(account).toBeDefined();
  expect(account?.balance).toBeGreaterThan(0);
});\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// TESTING SUMMARY
// =============================================================================

console.log('TYPESCRIPT TESTING SUMMARY:\n');

console.log('Configuration:');
console.log('  • ts-jest preset');
console.log('  • Path mapping support');
console.log('  • Coverage thresholds');
console.log('  • Type declarations (@types/jest)\n');

console.log('Test Structure:');
console.log('  • Typed test suites');
console.log('  • beforeEach/afterEach setup');
console.log('  • describe/it organization');
console.log('  • Type-safe assertions\n');

console.log('Mocking:');
console.log('  • jest.Mocked<T> type');
console.log('  • Partial mocks');
console.log('  • Mock return types');
console.log('  • Mock implementations\n');

console.log('Best Practices:');
console.log('  ✓ Type all test data');
console.log('  ✓ Use factory functions');
console.log('  ✓ Create test builders');
console.log('  ✓ Type-safe mocks');
console.log('  ✓ Custom matchers');
console.log('  ✓ Async utilities');
console.log('  ✓ Clear test naming');
console.log('  ✓ AAA pattern (Arrange-Act-Assert)');

console.log();
```

---

## Question 14: Explain TypeScript performance optimization and best practices

### Answer:

**Performance Optimization** in TypeScript involves compiler settings, code patterns, and architectural decisions that improve build times, runtime performance, and developer experience.

### Key Areas:
1. **Compiler Performance** - Fast compilation
2. **Type Performance** - Efficient types
3. **Runtime Optimization** - Production code
4. **Bundle Size** - Smaller output
5. **Development Experience** - Fast feedback

### Banking Scenario: Performance Optimization at Emirates NBD

```typescript
console.log('=== TypeScript Performance Optimization - Emirates NBD ===\n');

// =============================================================================
// 1. COMPILER PERFORMANCE
// =============================================================================

console.log('1. COMPILER PERFORMANCE:\n');

console.log('Optimized tsconfig.json:\n');
console.log(`{
  "compilerOptions": {
    // Skip type checking of declaration files
    "skipLibCheck": true,
    
    // Faster incremental builds
    "incremental": true,
    "tsBuildInfoFile": "./.tsbuildinfo",
    
    // Only emit changed files
    "assumeChangesOnlyAffectDirectDependencies": true,
    
    // Faster module resolution
    "moduleResolution": "node",
    "resolveJsonModule": true,
    
    // Don't emit on error
    "noEmitOnError": true,
    
    // Disable unused checks in development
    "noUnusedLocals": false,      // Enable in CI
    "noUnusedParameters": false,   // Enable in CI
    
    // Use project references for monorepos
    "composite": true
  }
}\n`);

console.log('Project References (Monorepo):');
console.log('  • Split large projects into smaller units');
console.log('  • Parallel compilation');
console.log('  • Only rebuild changed projects');
console.log('  • Better IDE performance\n');

console.log('Build Commands:');
console.log('  tsc --build --incremental    # Incremental build');
console.log('  tsc --build --watch          # Watch mode');
console.log('  tsc --build --force          # Full rebuild\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPE PERFORMANCE
// =============================================================================

console.log('2. TYPE PERFORMANCE:\n');

console.log('❌ Avoid - Complex recursive types:\n');
console.log(`type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object
    ? DeepReadonly<T[P]>
    : T[P];
};

// Can be slow for deeply nested objects\n`);

console.log('✅ Prefer - Simpler types:\n');
console.log(`type ReadonlyAccount = Readonly<Account>;

// Or use library types
import { DeepReadonly } from 'utility-types';\n`);

console.log('❌ Avoid - Union of many literals:\n');
console.log(`type AccountNumber =
  | 'ACC-0001' | 'ACC-0002' | 'ACC-0003'
  // ... 1000 more
  | 'ACC-1000';

// Slow type checking\n`);

console.log('✅ Prefer - String patterns:\n');
console.log(`type AccountNumber = \`ACC-\${number}\`;

// Or branded types
type AccountNumber = string & { __brand: 'AccountNumber' };\n`);

console.log('Type Inference vs Explicit Types:\n');
console.log(`
// ❌ Over-annotation
const accounts: Account[] = getAccounts();

// ✅ Let TypeScript infer
const accounts = getAccounts(); // Inferred as Account[]

// ✅ Explicit only when needed
const accounts: Account[] = []; // Empty array needs annotation\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. RUNTIME OPTIMIZATION
// =============================================================================

console.log('3. RUNTIME OPTIMIZATION:\n');

// Efficient object creation
console.log('Object Creation:\n');

console.log('❌ Avoid - Object spread in loops:\n');
console.log(`const accounts = [];
for (const data of accountData) {
  accounts.push({ ...data, processed: true }); // Creates many objects
}\n`);

console.log('✅ Prefer - Direct mutation or map:\n');
console.log(`const accounts = accountData.map(data => ({
  ...data,
  processed: true
}));

// Or mutate if appropriate
accountData.forEach(data => {
  data.processed = true;
});\n`);

// Type guards efficiency
console.log('Type Guards:\n');

interface Account {
    id: string;
    type: 'SAVINGS' | 'CURRENT';
    balance: number;
}

console.log('❌ Avoid - Complex type guards:\n');
console.log(`function isAccount(obj: any): obj is Account {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    typeof obj.id === 'string' &&
    ['SAVINGS', 'CURRENT'].includes(obj.type) &&
    typeof obj.balance === 'number' &&
    obj.balance >= 0
  );
}\n`);

console.log('✅ Prefer - Simple checks + validation library:\n');
console.log(`import { z } from 'zod';

const AccountSchema = z.object({
  id: z.string(),
  type: z.enum(['SAVINGS', 'CURRENT']),
  balance: z.number().min(0)
});

function isAccount(obj: unknown): obj is Account {
  return AccountSchema.safeParse(obj).success;
}\n`);

// Memoization
console.log('Memoization:\n');

class AccountAnalytics {
    private cache = new Map<string, number>();
    
    calculateTotalBalance(accounts: Account[]): number {
        const cacheKey = accounts.map(a => a.id).join(',');
        
        if (this.cache.has(cacheKey)) {
            return this.cache.get(cacheKey)!;
        }
        
        const total = accounts.reduce((sum, acc) => sum + acc.balance, 0);
        this.cache.set(cacheKey, total);
        return total;
    }
}

console.log('Memoization pattern for expensive calculations:');
console.log('  • Cache results by key');
console.log('  • Clear cache when data changes');
console.log('  • Use Map for better performance\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. BUNDLE SIZE OPTIMIZATION
// =============================================================================

console.log('4. BUNDLE SIZE:\n');

console.log('Tree Shaking:\n');
console.log(`// ❌ Avoid - Import everything
import * as _ from 'lodash';

// ✅ Prefer - Import only what you need
import { debounce, throttle } from 'lodash';

// ✅ Even better - Use lodash-es
import debounce from 'lodash-es/debounce';\n`);

console.log('Code Splitting:\n');
console.log(`// Dynamic imports for large modules
async function processLargeReport() {
  const { generateReport } = await import('./reporting');
  return generateReport();
}

// Lazy load features
const AdminPanel = lazy(() => import('./AdminPanel'));\n`);

console.log('Remove Development Code:\n');
console.log(`// Use environment variables
if (process.env.NODE_ENV === 'development') {
  console.log('Debug info'); // Removed in production
}

// Tree-shakeable debug utilities
export const debug = 
  process.env.NODE_ENV === 'development'
    ? console.log
    : () => {};\n`);

console.log('Production Build:');
console.log(`{
  "compilerOptions": {
    "removeComments": true,    // Remove comments
    "sourceMap": false,        // No source maps
    "declaration": false       // No .d.ts files
  }
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. DEVELOPMENT EXPERIENCE
// =============================================================================

console.log('5. DEVELOPMENT EXPERIENCE:\n');

console.log('Fast Feedback Loop:\n');
console.log(`// Use ts-node-dev for development
npm install -D ts-node-dev

// package.json
"scripts": {
  "dev": "ts-node-dev --respawn --transpile-only src/server.ts"
}

// --transpile-only: Skip type checking (faster)
// --respawn: Restart on changes\n`);

console.log('Separate Type Checking:\n');
console.log(`// package.json
"scripts": {
  "dev": "ts-node-dev --transpile-only src/server.ts",
  "type-check": "tsc --noEmit",
  "type-check:watch": "tsc --noEmit --watch"
}

// Run type checking in parallel during development\n`);

console.log('Editor Performance:\n');
console.log('  • Use .gitignore for node_modules');
console.log('  • Exclude dist/ from search');
console.log('  • Use incremental compilation');
console.log('  • Split large files');
console.log('  • Restart TS server occasionally\n');

console.log('CI/CD Optimization:\n');
console.log(`// GitHub Actions
- name: Cache dependencies
  uses: actions/cache@v2
  with:
    path: ~/.npm
    key: \${{ runner.os }}-node-\${{ hashFiles('**/package-lock.json') }}

- name: Cache TypeScript build
  uses: actions/cache@v2
  with:
    path: .tsbuildinfo
    key: \${{ runner.os }}-tsbuild-\${{ hashFiles('**/*.ts') }}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// BEST PRACTICES SUMMARY
// =============================================================================

console.log('PERFORMANCE BEST PRACTICES:\n');

console.log('Compilation:');
console.log('  ✓ Enable skipLibCheck');
console.log('  ✓ Use incremental builds');
console.log('  ✓ Project references for monorepos');
console.log('  ✓ Only build changed files');
console.log('  ✓ Parallel type checking\n');

console.log('Type System:');
console.log('  ✓ Avoid complex recursive types');
console.log('  ✓ Use simpler union types');
console.log('  ✓ Leverage type inference');
console.log('  ✓ Import only needed types');
console.log('  ✓ Use utility types from libraries\n');

console.log('Runtime:');
console.log('  ✓ Minimize object creation');
console.log('  ✓ Use Map/Set for lookups');
console.log('  ✓ Memoize expensive operations');
console.log('  ✓ Lazy load large modules');
console.log('  ✓ Validate at boundaries\n');

console.log('Bundle:');
console.log('  ✓ Tree shaking');
console.log('  ✓ Code splitting');
console.log('  ✓ Dynamic imports');
console.log('  ✓ Remove dev code');
console.log('  ✓ Minimize dependencies\n');

console.log('Development:');
console.log('  ✓ Fast restart (ts-node-dev)');
console.log('  ✓ Separate type checking');
console.log('  ✓ Watch mode');
console.log('  ✓ Cache in CI/CD');
console.log('  ✓ Optimize editor settings\n');

console.log('Monitoring:');
console.log('  • Measure build times');
console.log('  • Track bundle sizes');
console.log('  • Profile compilation');
console.log('  • Monitor IDE performance');

console.log();
```

### Key Takeaways:

**Design Patterns**:
- Singleton for shared resources
- Factory for object creation
- Observer for event systems
- Repository for data access
- Builder for complex objects

**Async Operations**:
- Promise<T> typing
- async/await patterns
- Error handling with types
- Promise combinators
- Advanced patterns (retry, timeout, queue)

**Testing**:
- Jest with ts-jest
- Typed test suites
- Type-safe mocks
- Custom matchers
- Test utilities

**Performance**:
- Compiler optimization
- Efficient type patterns
- Runtime optimization
- Bundle size reduction
- Development experience improvements

---

**End of Questions 11-14**
