# Dependency Inversion Principle (DIP)

## What is Dependency Inversion Principle?

**Definition**: High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions.

**Simple Explanation**: Don't depend on concrete classes directly. Depend on interfaces or abstract classes. This inverts the traditional dependency flow where high-level code depends on low-level code.

**Robert C. Martin's Definition**:
> A. High-level modules should not depend on low-level modules. Both should depend on abstractions.
> 
> B. Abstractions should not depend on details. Details should depend on abstractions.

## Why is it Important?

### Benefits:
1. **Loose Coupling** - Components are independent
2. **Easy Testing** - Can mock dependencies easily
3. **Flexibility** - Swap implementations without changing code
4. **Parallel Development** - Teams work independently
5. **Better Architecture** - Clear separation of concerns
6. **Easier Maintenance** - Changes localized to one area

### Problems Without DIP:
- **Tight Coupling** - High-level code tied to low-level details
- **Hard to Test** - Can't replace dependencies with mocks
- **Inflexible** - Changing implementation breaks everything
- **Difficult to Extend** - Adding features requires many changes
- **Vendor Lock-in** - Stuck with specific implementations
- **Code Duplication** - Similar code in multiple places

---

## Banking Domain Examples

### ❌ BAD Example - Violating DIP

```typescript
console.log('=== DIP Violation - Emirates NBD Banking ===\n');

// ❌ LOW-LEVEL: Concrete implementation classes
class MySQLDatabase {
    connect(): void {
        console.log('❌ Connecting to MySQL database...');
    }
    
    saveAccount(account: any): void {
        console.log(`❌ Saving account to MySQL: ${JSON.stringify(account)}`);
    }
    
    findAccount(accountNumber: string): any {
        console.log(`❌ Finding account in MySQL: ${accountNumber}`);
        return { accountNumber, balance: 5000 };
    }
}

class EmailService {
    sendEmail(to: string, subject: string, body: string): void {
        console.log(`❌ Sending email via SMTP:`);
        console.log(`   To: ${to}`);
        console.log(`   Subject: ${subject}`);
    }
}

class AuditLogger {
    log(message: string): void {
        console.log(`❌ [AUDIT] ${message}`);
    }
}

class SMSService {
    sendSMS(phoneNumber: string, message: string): void {
        console.log(`❌ Sending SMS via Twilio to ${phoneNumber}`);
    }
}

// ❌ HIGH-LEVEL: Directly depends on concrete classes
class ViolatedAccountService {
    // ❌ Tight coupling to specific implementations
    private database: MySQLDatabase;
    private emailService: EmailService;
    private auditLogger: AuditLogger;
    private smsService: SMSService;
    
    constructor() {
        // ❌ Creating dependencies directly
        this.database = new MySQLDatabase();
        this.emailService = new EmailService();
        this.auditLogger = new AuditLogger();
        this.smsService = new SMSService();
    }
    
    createAccount(accountNumber: string, customerEmail: string, phone: string): void {
        console.log(`\nCreating account ${accountNumber}...`);
        
        // ❌ Directly using concrete implementations
        this.database.connect();
        this.database.saveAccount({ accountNumber, balance: 0 });
        
        this.auditLogger.log(`Account created: ${accountNumber}`);
        
        this.emailService.sendEmail(
            customerEmail,
            'Account Created',
            'Your account has been created'
        );
        
        this.smsService.sendSMS(phone, 'Account created successfully');
    }
    
    getAccount(accountNumber: string): any {
        this.database.connect();
        return this.database.findAccount(accountNumber);
    }
}

console.log('PROBLEMS with this design:\n');
console.log('❌ Cannot switch from MySQL to PostgreSQL without changing AccountService');
console.log('❌ Cannot switch from email to push notifications without changing code');
console.log('❌ Cannot test AccountService without real database/email/SMS');
console.log('❌ AccountService knows too much about low-level details');
console.log('❌ If MySQL API changes, AccountService must change');
console.log('❌ Difficult to add new notification channels');
console.log('❌ Tight coupling = fragile code\n');

// Demonstration of the problems
const violatedService = new ViolatedAccountService();
violatedService.createAccount('ACC-001', 'customer@email.com', '+971501234567');

console.log('\n' + '='.repeat(70) + '\n');
```

---

### ✅ GOOD Example - Following DIP

```typescript
console.log('=== DIP Compliant Design - Emirates NBD Banking ===\n');

// ============================================================
// STEP 1: Define Abstractions (Interfaces)
// ============================================================

// Database abstraction
interface IAccountRepository {
    save(account: Account): Promise<void>;
    findByAccountNumber(accountNumber: string): Promise<Account | null>;
    update(account: Account): Promise<void>;
    delete(accountNumber: string): Promise<void>;
}

// Notification abstraction
interface INotificationService {
    send(recipient: string, message: string): Promise<void>;
    getChannelName(): string;
}

// Logging abstraction
interface ILogger {
    info(message: string): void;
    error(message: string): void;
    warn(message: string): void;
}

// Transaction abstraction
interface ITransactionProcessor {
    process(transaction: Transaction): Promise<boolean>;
}

// ============================================================
// STEP 2: Domain Models (High-level)
// ============================================================

class Account {
    constructor(
        public accountNumber: string,
        public customerName: string,
        public email: string,
        public phone: string,
        public balance: number = 0
    ) {}
}

class Transaction {
    constructor(
        public id: string,
        public accountNumber: string,
        public amount: number,
        public type: 'DEPOSIT' | 'WITHDRAWAL',
        public timestamp: Date = new Date()
    ) {}
}

// ============================================================
// STEP 3: High-Level Business Logic (Depends on abstractions)
// ============================================================

class AccountService {
    // ✅ Depends on interfaces, not concrete classes
    constructor(
        private repository: IAccountRepository,
        private notifications: INotificationService[],
        private logger: ILogger,
        private transactionProcessor: ITransactionProcessor
    ) {}
    
    async createAccount(
        accountNumber: string,
        customerName: string,
        email: string,
        phone: string
    ): Promise<Account> {
        this.logger.info(`Creating account: ${accountNumber}`);
        
        const account = new Account(accountNumber, customerName, email, phone);
        
        // ✅ Using abstraction - doesn't know if it's MySQL, MongoDB, etc.
        await this.repository.save(account);
        
        this.logger.info(`Account created: ${accountNumber}`);
        
        // ✅ Notify via all channels - doesn't know if email, SMS, push, etc.
        const message = `Dear ${customerName}, your account ${accountNumber} has been created!`;
        for (const notifier of this.notifications) {
            await notifier.send(email, message);
            this.logger.info(`Notification sent via ${notifier.getChannelName()}`);
        }
        
        return account;
    }
    
    async deposit(accountNumber: string, amount: number): Promise<void> {
        this.logger.info(`Deposit: ${amount} to ${accountNumber}`);
        
        // ✅ Using abstraction
        const account = await this.repository.findByAccountNumber(accountNumber);
        if (!account) {
            throw new Error('Account not found');
        }
        
        const transaction = new Transaction(
            `TXN-${Date.now()}`,
            accountNumber,
            amount,
            'DEPOSIT'
        );
        
        // ✅ Process transaction via abstraction
        const success = await this.transactionProcessor.process(transaction);
        
        if (success) {
            account.balance += amount;
            await this.repository.update(account);
            
            const message = `Deposited ${amount} AED. New balance: ${account.balance} AED`;
            for (const notifier of this.notifications) {
                await notifier.send(account.email, message);
            }
        }
    }
    
    async getAccount(accountNumber: string): Promise<Account | null> {
        return await this.repository.findByAccountNumber(accountNumber);
    }
}

// ============================================================
// STEP 4: Low-Level Implementations (Depend on abstractions)
// ============================================================

// ✅ Repository Implementation 1: MySQL
class MySQLAccountRepository implements IAccountRepository {
    async save(account: Account): Promise<void> {
        console.log(`  ✓ [MySQL] Saving account to MySQL: ${account.accountNumber}`);
        // Actual MySQL code here
    }
    
    async findByAccountNumber(accountNumber: string): Promise<Account | null> {
        console.log(`  ✓ [MySQL] Finding account in MySQL: ${accountNumber}`);
        // Actual MySQL query here
        return new Account(accountNumber, 'John Doe', 'john@email.com', '+971501234567', 5000);
    }
    
    async update(account: Account): Promise<void> {
        console.log(`  ✓ [MySQL] Updating account in MySQL: ${account.accountNumber}`);
    }
    
    async delete(accountNumber: string): Promise<void> {
        console.log(`  ✓ [MySQL] Deleting account from MySQL: ${accountNumber}`);
    }
}

// ✅ Repository Implementation 2: MongoDB
class MongoDBAccountRepository implements IAccountRepository {
    async save(account: Account): Promise<void> {
        console.log(`  ✓ [MongoDB] Saving account to MongoDB: ${account.accountNumber}`);
    }
    
    async findByAccountNumber(accountNumber: string): Promise<Account | null> {
        console.log(`  ✓ [MongoDB] Finding account in MongoDB: ${accountNumber}`);
        return new Account(accountNumber, 'Jane Smith', 'jane@email.com', '+971509876543', 3000);
    }
    
    async update(account: Account): Promise<void> {
        console.log(`  ✓ [MongoDB] Updating account in MongoDB: ${account.accountNumber}`);
    }
    
    async delete(accountNumber: string): Promise<void> {
        console.log(`  ✓ [MongoDB] Deleting account from MongoDB: ${accountNumber}`);
    }
}

// ✅ Notification Implementation 1: Email
class EmailNotificationService implements INotificationService {
    getChannelName(): string {
        return 'Email';
    }
    
    async send(recipient: string, message: string): Promise<void> {
        console.log(`  ✓ [Email] Sending to ${recipient}`);
        console.log(`     Message: ${message}`);
    }
}

// ✅ Notification Implementation 2: SMS
class SMSNotificationService implements INotificationService {
    getChannelName(): string {
        return 'SMS';
    }
    
    async send(recipient: string, message: string): Promise<void> {
        console.log(`  ✓ [SMS] Sending to ${recipient}`);
        console.log(`     Message: ${message}`);
    }
}

// ✅ Notification Implementation 3: Push
class PushNotificationService implements INotificationService {
    getChannelName(): string {
        return 'Push Notification';
    }
    
    async send(recipient: string, message: string): Promise<void> {
        console.log(`  ✓ [Push] Sending to ${recipient}`);
        console.log(`     Message: ${message}`);
    }
}

// ✅ Notification Implementation 4: WhatsApp
class WhatsAppNotificationService implements INotificationService {
    getChannelName(): string {
        return 'WhatsApp';
    }
    
    async send(recipient: string, message: string): Promise<void> {
        console.log(`  ✓ [WhatsApp] Sending to ${recipient}`);
        console.log(`     Message: ${message}`);
    }
}

// ✅ Logger Implementation 1: Console
class ConsoleLogger implements ILogger {
    info(message: string): void {
        console.log(`  ℹ️  [INFO] ${message}`);
    }
    
    error(message: string): void {
        console.log(`  ❌ [ERROR] ${message}`);
    }
    
    warn(message: string): void {
        console.log(`  ⚠️  [WARN] ${message}`);
    }
}

// ✅ Logger Implementation 2: File
class FileLogger implements ILogger {
    info(message: string): void {
        console.log(`  ✓ [FileLogger] INFO: ${message}`);
        // Write to file
    }
    
    error(message: string): void {
        console.log(`  ✓ [FileLogger] ERROR: ${message}`);
    }
    
    warn(message: string): void {
        console.log(`  ✓ [FileLogger] WARN: ${message}`);
    }
}

// ✅ Transaction Processor Implementation
class DefaultTransactionProcessor implements ITransactionProcessor {
    async process(transaction: Transaction): Promise<boolean> {
        console.log(`  ✓ [Transaction] Processing ${transaction.type}: ${transaction.amount} AED`);
        console.log(`     Transaction ID: ${transaction.id}`);
        // Validation, fraud detection, etc.
        return true;
    }
}

// ============================================================
// STEP 5: Dependency Injection (Composition Root)
// ============================================================

console.log('1️⃣  Configuration 1: MySQL + Email + SMS + Console Logger\n');

const config1 = {
    repository: new MySQLAccountRepository(),
    notifications: [
        new EmailNotificationService(),
        new SMSNotificationService()
    ],
    logger: new ConsoleLogger(),
    transactionProcessor: new DefaultTransactionProcessor()
};

const service1 = new AccountService(
    config1.repository,
    config1.notifications,
    config1.logger,
    config1.transactionProcessor
);

await service1.createAccount('ACC-001', 'Ahmed Al Mansoori', 'ahmed@email.com', '+971501234567');
await service1.deposit('ACC-001', 1000);

console.log('\n' + '─'.repeat(70) + '\n');

// ✅ Switch to different implementations - NO CODE CHANGE in AccountService!
console.log('2️⃣  Configuration 2: MongoDB + Email + Push + WhatsApp + File Logger\n');

const config2 = {
    repository: new MongoDBAccountRepository(), // Changed!
    notifications: [
        new EmailNotificationService(),
        new PushNotificationService(), // Added!
        new WhatsAppNotificationService() // Added!
    ],
    logger: new FileLogger(), // Changed!
    transactionProcessor: new DefaultTransactionProcessor()
};

const service2 = new AccountService(
    config2.repository,
    config2.notifications,
    config2.logger,
    config2.transactionProcessor
);

await service2.createAccount('ACC-002', 'Fatima Hassan', 'fatima@email.com', '+971509876543');

console.log('\n' + '='.repeat(70));
console.log('DIP Benefits Demonstrated:');
console.log('  ✅ Switched MySQL → MongoDB: NO AccountService code change');
console.log('  ✅ Added Push/WhatsApp: NO AccountService code change');
console.log('  ✅ Switched Console → File Logger: NO AccountService code change');
console.log('  ✅ AccountService only knows about interfaces');
console.log('  ✅ Can easily test with mock implementations');
console.log('  ✅ High-level business logic independent of low-level details');
console.log('='.repeat(70) + '\n');
```

---

## Dependency Injection Patterns

```typescript
console.log('=== Dependency Injection Patterns ===\n');

// Pattern 1: Constructor Injection (Recommended)
console.log('1. CONSTRUCTOR INJECTION:\n');

class PaymentService {
    constructor(
        private gateway: IPaymentGateway,
        private logger: ILogger
    ) {}
    
    processPayment(amount: number): void {
        this.logger.info(`Processing payment: ${amount} AED`);
        this.gateway.charge(amount);
    }
}

interface IPaymentGateway {
    charge(amount: number): void;
}

class StripeGateway implements IPaymentGateway {
    charge(amount: number): void {
        console.log(`  ✓ Charged ${amount} AED via Stripe`);
    }
}

const paymentService = new PaymentService(
    new StripeGateway(),
    new ConsoleLogger()
);
paymentService.processPayment(500);

console.log('  ✅ Dependencies clear from constructor');
console.log('  ✅ Immutable after construction');
console.log('  ✅ Easy to test\n');

// Pattern 2: Property Injection (Use sparingly)
console.log('2. PROPERTY INJECTION:\n');

class ReportGenerator {
    logger?: ILogger; // Optional dependency
    
    generate(): void {
        console.log('  ✓ Generating report...');
        this.logger?.info('Report generated');
    }
}

const reportGen = new ReportGenerator();
reportGen.logger = new ConsoleLogger(); // Set later
reportGen.generate();

console.log('  ⚠️  Use only for optional dependencies');
console.log('  ⚠️  Can be null - need null checks\n');

// Pattern 3: Method Injection
console.log('3. METHOD INJECTION:\n');

class DocumentPrinter {
    print(document: string, printer: IPrinter): void {
        console.log(`  ✓ Printing document: ${document}`);
        printer.print(document);
    }
}

interface IPrinter {
    print(content: string): void;
}

class PDFPrinter implements IPrinter {
    print(content: string): void {
        console.log(`  ✓ Printed as PDF: ${content}`);
    }
}

const docPrinter = new DocumentPrinter();
docPrinter.print('Invoice', new PDFPrinter());

console.log('  ✅ Different printer per call');
console.log('  ✅ Flexibility for each operation\n');

console.log('='.repeat(70) + '\n');
```

---

## DI Container Example

```typescript
console.log('=== Dependency Injection Container ===\n');

// Simple DI Container for automatic dependency resolution
class DIContainer {
    private services = new Map<string, any>();
    
    register<T>(name: string, instance: T): void {
        this.services.set(name, instance);
    }
    
    resolve<T>(name: string): T {
        const service = this.services.get(name);
        if (!service) {
            throw new Error(`Service not found: ${name}`);
        }
        return service;
    }
}

// Setup container
console.log('Setting up DI Container:\n');

const container = new DIContainer();

// Register dependencies
container.register<IAccountRepository>('AccountRepository', new MySQLAccountRepository());
container.register<ILogger>('Logger', new ConsoleLogger());
container.register<INotificationService[]>('Notifications', [
    new EmailNotificationService(),
    new SMSNotificationService()
]);
container.register<ITransactionProcessor>('TransactionProcessor', new DefaultTransactionProcessor());

console.log('  ✓ Registered AccountRepository');
console.log('  ✓ Registered Logger');
console.log('  ✓ Registered Notifications (Email, SMS)');
console.log('  ✓ Registered TransactionProcessor\n');

// Resolve and use
const accountService = new AccountService(
    container.resolve<IAccountRepository>('AccountRepository'),
    container.resolve<INotificationService[]>('Notifications'),
    container.resolve<ILogger>('Logger'),
    container.resolve<ITransactionProcessor>('TransactionProcessor')
);

console.log('Application Running with Injected Dependencies:\n');
await accountService.createAccount('ACC-100', 'Container Test', 'test@email.com', '+971501111111');

console.log('\n' + '='.repeat(70) + '\n');
```

---

## Testing with DIP

```typescript
console.log('=== Testing Benefits of DIP ===\n');

// ✅ Mock implementations for testing
class MockAccountRepository implements IAccountRepository {
    private accounts = new Map<string, Account>();
    
    async save(account: Account): Promise<void> {
        this.accounts.set(account.accountNumber, account);
        console.log(`  🧪 [Mock] Saved account: ${account.accountNumber}`);
    }
    
    async findByAccountNumber(accountNumber: string): Promise<Account | null> {
        const account = this.accounts.get(accountNumber) || null;
        console.log(`  🧪 [Mock] Found account: ${accountNumber} - ${account ? 'Yes' : 'No'}`);
        return account;
    }
    
    async update(account: Account): Promise<void> {
        this.accounts.set(account.accountNumber, account);
        console.log(`  🧪 [Mock] Updated account: ${account.accountNumber}`);
    }
    
    async delete(accountNumber: string): Promise<void> {
        this.accounts.delete(accountNumber);
        console.log(`  🧪 [Mock] Deleted account: ${accountNumber}`);
    }
}

class MockNotificationService implements INotificationService {
    sentMessages: Array<{ recipient: string, message: string }> = [];
    
    getChannelName(): string {
        return 'Mock';
    }
    
    async send(recipient: string, message: string): Promise<void> {
        this.sentMessages.push({ recipient, message });
        console.log(`  🧪 [Mock] Notification queued for ${recipient}`);
    }
}

class MockLogger implements ILogger {
    logs: string[] = [];
    
    info(message: string): void {
        this.logs.push(`INFO: ${message}`);
    }
    
    error(message: string): void {
        this.logs.push(`ERROR: ${message}`);
    }
    
    warn(message: string): void {
        this.logs.push(`WARN: ${message}`);
    }
}

// Test with mocks
console.log('Running Tests with Mock Dependencies:\n');

const mockRepo = new MockAccountRepository();
const mockNotifier = new MockNotificationService();
const mockLogger = new MockLogger();
const mockProcessor = new DefaultTransactionProcessor();

const testService = new AccountService(
    mockRepo,
    [mockNotifier],
    mockLogger,
    mockProcessor
);

await testService.createAccount('TEST-001', 'Test User', 'test@test.com', '+971500000000');
await testService.deposit('TEST-001', 500);

console.log('\n📊 Test Results:');
console.log(`  ✓ Logs captured: ${mockLogger.logs.length}`);
console.log(`  ✓ Notifications sent: ${mockNotifier.sentMessages.length}`);
console.log(`  ✓ No real database or email service used`);
console.log(`  ✓ Fast, isolated, repeatable tests`);

console.log('\n' + '='.repeat(70) + '\n');
```

---

## Interview Questions & Answers

### Q1: What is Dependency Inversion Principle?
**Answer**: DIP has two parts:
1. High-level modules shouldn't depend on low-level modules - both should depend on abstractions
2. Abstractions shouldn't depend on details - details should depend on abstractions

**Example**: AccountService (high-level) shouldn't depend on MySQLDatabase (low-level). Both should depend on IRepository (abstraction).

### Q2: What's the difference between DIP and Dependency Injection?
**Answer**:
- **DIP** = Design principle (depend on abstractions, not concretions)
- **Dependency Injection (DI)** = Design pattern (how to provide dependencies)

DI is a technique to achieve DIP:
```typescript
// DIP: Depend on interface
class Service {
    constructor(private repo: IRepository) {} // Abstraction
}

// DI: Inject concrete implementation
const service = new Service(new MySQLRepository()); // Injection
```

### Q3: Give a banking example of DIP violation and solution
**Answer**:
```typescript
// ❌ Violation - Direct dependency on concrete class
class AccountService {
    private db = new MySQLDatabase(); // Tight coupling!
    
    save(account) {
        this.db.insert(account); // Directly using MySQL
    }
}

// ✅ Solution - Depend on abstraction
interface IDatabase {
    insert(data: any): void;
}

class AccountService {
    constructor(private db: IDatabase) {} // Abstraction!
    
    save(account) {
        this.db.insert(account); // Works with any DB
    }
}

// Inject implementation
const service = new AccountService(new MySQLDatabase());
// Or: new AccountService(new MongoDatabase());
```

### Q4: What are the benefits of DIP in testing?
**Answer**:
1. **Easy Mocking** - Replace real dependencies with mocks
2. **No External Dependencies** - No real database/email/API needed
3. **Fast Tests** - In-memory implementations
4. **Isolated Tests** - Test one component at a time
5. **Repeatable** - No side effects

```typescript
// Testing without DIP - HARD
class Service {
    private db = new RealDatabase(); // Can't replace!
    // Tests will hit real database
}

// Testing with DIP - EASY
class Service {
    constructor(private db: IDatabase) {}
}

const testService = new Service(new MockDatabase()); // Easy!
```

### Q5: How do you implement Dependency Injection?
**Answer**: Three main patterns:

**1. Constructor Injection** (Best for required dependencies)
```typescript
class Service {
    constructor(private repo: IRepository) {} // Injected
}
```

**2. Property Injection** (For optional dependencies)
```typescript
class Service {
    logger?: ILogger; // Set later
}
service.logger = new ConsoleLogger();
```

**3. Method Injection** (For per-call dependencies)
```typescript
class Service {
    process(data, validator: IValidator) {} // Injected per call
}
```

### Q6: What is a DI Container?
**Answer**: A DI Container (IoC Container) automatically manages dependencies:

```typescript
// Manual DI - tedious
const logger = new Logger();
const db = new Database();
const emailer = new Emailer();
const service = new Service(db, logger, emailer);

// DI Container - automatic
container.register('Logger', Logger);
container.register('Database', Database);
container.register('Emailer', Emailer);
container.register('Service', Service);

const service = container.resolve('Service'); // Auto-wired!
```

Popular containers: InversifyJS, TSyringe, Awilix (TypeScript)

### Q7: When should you use DIP?
**Answer**: Use DIP when:
1. **Need flexibility** - Switch implementations (MySQL → PostgreSQL)
2. **Need testability** - Replace with mocks
3. **Working in layers** - Business logic shouldn't know about database
4. **External dependencies** - Third-party APIs, file systems
5. **Multiple implementations** - Different payment gateways

Don't over-engineer simple cases:
```typescript
// ❌ Overkill for simple utility
interface IStringReverser {
    reverse(str: string): string;
}

// ✅ Just use a simple function
function reverse(str: string): string {
    return str.split('').reverse().join('');
}
```

---

## Key Takeaways

### ✅ DO:
- Depend on interfaces, not concrete classes
- Use dependency injection for providing implementations
- Design abstractions from consumer's perspective
- Keep abstractions stable (don't change often)
- Use DI container for complex dependency graphs
- Write against interfaces in high-level code

### ❌ DON'T:
- Create dependencies with `new` inside classes
- Reference concrete classes in high-level modules
- Make abstractions depend on implementation details
- Create interface for every class (use when needed)
- Expose implementation details in interfaces
- Skip DIP for simple utilities

### Remember:
> "Depend upon abstractions, not concretions"
> 
> If you can't easily test a class by mocking its dependencies, you're violating DIP!

---

## Summary

**Dependency Inversion Principle** creates flexible, testable banking applications:

- ✅ AccountService depends on IRepository interface, not MySQLRepository
- ✅ Inject implementations via constructor (Dependency Injection)
- ✅ Switch MySQL → MongoDB without changing AccountService
- ✅ Add new notification channels without modifying business logic
- ✅ Test with mock implementations instead of real database

Following DIP makes your banking application:
- **Flexible** - Swap implementations easily
- **Testable** - Mock dependencies in tests
- **Maintainable** - Changes localized
- **Scalable** - Add features without breaking existing code
- **Independent** - High-level logic doesn't know low-level details

**The key question**: "Can I easily replace this dependency with a different implementation?"

If NO, apply DIP:
1. Extract interface from concrete class
2. Make high-level code depend on interface
3. Inject concrete implementation via constructor
4. Use DI container for complex scenarios

**Real-world impact**:
- Switch databases without code changes
- Add new notification channels instantly
- Test with mocks - no real database needed
- Different configurations for dev/test/prod
- Parallel development - teams work independently
- No vendor lock-in - switch providers easily

**Architecture Flow**:
```
High-Level (Business Logic)
         ↓ depends on
    Abstractions (Interfaces)
         ↑ implemented by
Low-Level (Implementation Details)
```

This is "inversion" - normally high-level depends on low-level, but we invert it!
