# Single Responsibility Principle (SRP)

## What is Single Responsibility Principle?

**Definition**: A class should have only ONE reason to change, meaning it should have only ONE responsibility or job.

**Simple Explanation**: Each class should do ONE thing and do it well. If a class is doing multiple things, it should be split into separate classes.

## Why is it Important?

### Benefits:
1. **Easier to Understand** - Code is clearer when each class has a single purpose
2. **Easier to Maintain** - Changes to one responsibility don't affect others
3. **Easier to Test** - Testing a single responsibility is simpler
4. **Better Reusability** - Single-purpose classes are more reusable
5. **Reduced Coupling** - Classes are less dependent on each other
6. **Flexibility** - Easy to modify or replace individual responsibilities

### Problems Without SRP:
- **God Objects** - Classes that do everything (hard to maintain)
- **Fragile Code** - One change breaks multiple features
- **Difficult Testing** - Too many dependencies to mock
- **Poor Reusability** - Can't reuse without bringing unnecessary baggage
- **Merge Conflicts** - Multiple developers changing the same class

---

## Banking Domain Examples

### ❌ BAD Example - Violating SRP

```typescript
console.log('=== SRP Violation - Emirates NBD Banking ===\n');

// This class has TOO MANY responsibilities!
class AccountManager {
    private accounts: Map<string, any> = new Map();
    
    // Responsibility 1: Account Management
    createAccount(accountNumber: string, balance: number): void {
        const account = {
            accountNumber,
            balance,
            createdAt: new Date()
        };
        this.accounts.set(accountNumber, account);
        console.log(`Account created: ${accountNumber}`);
    }
    
    // Responsibility 2: Balance Operations
    deposit(accountNumber: string, amount: number): void {
        const account = this.accounts.get(accountNumber);
        if (account) {
            account.balance += amount;
            console.log(`Deposited ${amount} AED to ${accountNumber}`);
        }
    }
    
    withdraw(accountNumber: string, amount: number): void {
        const account = this.accounts.get(accountNumber);
        if (account && account.balance >= amount) {
            account.balance -= amount;
            console.log(`Withdrawn ${amount} AED from ${accountNumber}`);
        }
    }
    
    // Responsibility 3: Interest Calculation
    calculateInterest(accountNumber: string, rate: number): number {
        const account = this.accounts.get(accountNumber);
        if (account) {
            const interest = account.balance * (rate / 100);
            console.log(`Interest calculated: ${interest} AED`);
            return interest;
        }
        return 0;
    }
    
    // Responsibility 4: Sending Notifications
    sendEmailNotification(accountNumber: string, message: string): void {
        console.log(`📧 Email sent to account ${accountNumber}: ${message}`);
    }
    
    sendSMSNotification(accountNumber: string, message: string): void {
        console.log(`📱 SMS sent to account ${accountNumber}: ${message}`);
    }
    
    // Responsibility 5: Report Generation
    generateAccountStatement(accountNumber: string): string {
        const account = this.accounts.get(accountNumber);
        if (account) {
            const statement = `
=== Account Statement ===
Account: ${account.accountNumber}
Balance: ${account.balance} AED
Created: ${account.createdAt.toLocaleDateString()}
========================`;
            console.log(statement);
            return statement;
        }
        return '';
    }
    
    // Responsibility 6: Database Operations
    saveToDatabase(accountNumber: string): void {
        const account = this.accounts.get(accountNumber);
        console.log(`💾 Saving account ${accountNumber} to database...`);
        // Database logic here
    }
    
    loadFromDatabase(accountNumber: string): void {
        console.log(`📂 Loading account ${accountNumber} from database...`);
        // Database logic here
    }
    
    // Responsibility 7: Logging
    logTransaction(accountNumber: string, type: string, amount: number): void {
        console.log(`📝 LOG: [${new Date().toISOString()}] ${type} - ${amount} AED on ${accountNumber}`);
    }
}

// Problems with this design:
console.log('Problems with AccountManager:');
console.log('  ❌ Has 7 different responsibilities');
console.log('  ❌ Difficult to test (too many dependencies)');
console.log('  ❌ Hard to maintain (changes affect multiple areas)');
console.log('  ❌ Poor reusability (cant reuse notification logic separately)');
console.log('  ❌ Violates SRP completely!\n');

// Usage
const badAccountManager = new AccountManager();
badAccountManager.createAccount('ACC-001', 10000);
badAccountManager.deposit('ACC-001', 5000);
badAccountManager.calculateInterest('ACC-001', 2.5);
badAccountManager.sendEmailNotification('ACC-001', 'Deposit successful');
badAccountManager.generateAccountStatement('ACC-001');
badAccountManager.saveToDatabase('ACC-001');

console.log('\n' + '='.repeat(70) + '\n');
```

---

### ✅ GOOD Example - Following SRP

```typescript
console.log('=== SRP Compliant Design - Emirates NBD Banking ===\n');

// ============================================================
// 1. Account Entity - ONLY manages account data
// ============================================================
class Account {
    constructor(
        public accountNumber: string,
        public balance: number,
        public readonly createdAt: Date = new Date()
    ) {}
    
    getAccountInfo(): string {
        return `Account ${this.accountNumber}: ${this.balance} AED`;
    }
}

// ============================================================
// 2. Account Repository - ONLY handles data persistence
// ============================================================
class AccountRepository {
    private accounts: Map<string, Account> = new Map();
    
    save(account: Account): void {
        this.accounts.set(account.accountNumber, account);
        console.log(`💾 Account ${account.accountNumber} saved to database`);
    }
    
    findByAccountNumber(accountNumber: string): Account | undefined {
        return this.accounts.get(accountNumber);
    }
    
    findAll(): Account[] {
        return Array.from(this.accounts.values());
    }
    
    delete(accountNumber: string): void {
        this.accounts.delete(accountNumber);
        console.log(`🗑️  Account ${accountNumber} deleted from database`);
    }
}

// ============================================================
// 3. Transaction Service - ONLY handles money operations
// ============================================================
class TransactionService {
    deposit(account: Account, amount: number): void {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        account.balance += amount;
        console.log(`  ✓ Deposited ${amount} AED to ${account.accountNumber}`);
    }
    
    withdraw(account: Account, amount: number): void {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        if (account.balance < amount) {
            throw new Error('Insufficient funds');
        }
        account.balance -= amount;
        console.log(`  ✓ Withdrawn ${amount} AED from ${account.accountNumber}`);
    }
    
    transfer(from: Account, to: Account, amount: number): void {
        this.withdraw(from, amount);
        this.deposit(to, amount);
        console.log(`  ✓ Transferred ${amount} AED from ${from.accountNumber} to ${to.accountNumber}`);
    }
}

// ============================================================
// 4. Interest Calculator - ONLY calculates interest
// ============================================================
class InterestCalculator {
    calculateSimpleInterest(balance: number, rate: number, years: number = 1): number {
        return balance * (rate / 100) * years;
    }
    
    calculateCompoundInterest(balance: number, rate: number, years: number = 1, frequency: number = 12): number {
        return balance * Math.pow(1 + (rate / 100) / frequency, frequency * years) - balance;
    }
    
    calculateMonthlyInterest(balance: number, annualRate: number): number {
        return balance * (annualRate / 100) / 12;
    }
}

// ============================================================
// 5. Notification Service - ONLY sends notifications
// ============================================================
interface NotificationChannel {
    send(recipient: string, message: string): void;
}

class EmailNotificationService implements NotificationChannel {
    send(email: string, message: string): void {
        console.log(`  📧 Email to ${email}: ${message}`);
    }
}

class SMSNotificationService implements NotificationChannel {
    send(phone: string, message: string): void {
        console.log(`  📱 SMS to ${phone}: ${message}`);
    }
}

class PushNotificationService implements NotificationChannel {
    send(deviceId: string, message: string): void {
        console.log(`  🔔 Push to ${deviceId}: ${message}`);
    }
}

class NotificationService {
    private channels: NotificationChannel[] = [];
    
    addChannel(channel: NotificationChannel): void {
        this.channels.push(channel);
    }
    
    notifyAll(recipient: string, message: string): void {
        this.channels.forEach(channel => channel.send(recipient, message));
    }
}

// ============================================================
// 6. Statement Generator - ONLY generates reports
// ============================================================
class StatementGenerator {
    generateAccountStatement(account: Account): string {
        return `
╔════════════════════════════════════════╗
║        ACCOUNT STATEMENT               ║
╠════════════════════════════════════════╣
║ Account: ${account.accountNumber.padEnd(28)} ║
║ Balance: ${account.balance.toFixed(2).padEnd(20)} AED ║
║ Created: ${account.createdAt.toLocaleDateString().padEnd(28)} ║
╚════════════════════════════════════════╝`;
    }
    
    generateTransactionReport(transactions: Array<{type: string, amount: number, date: Date}>): string {
        let report = '\n╔════════════════════════════════════════╗\n';
        report += '║     TRANSACTION HISTORY                ║\n';
        report += '╠════════════════════════════════════════╣\n';
        
        transactions.forEach(txn => {
            const line = `║ ${txn.date.toLocaleDateString()} ${txn.type.padEnd(10)} ${txn.amount.toFixed(2).padStart(10)} AED ║`;
            report += line + '\n';
        });
        
        report += '╚════════════════════════════════════════╝';
        return report;
    }
}

// ============================================================
// 7. Audit Logger - ONLY logs activities
// ============================================================
class AuditLogger {
    log(level: 'INFO' | 'WARN' | 'ERROR', message: string): void {
        const timestamp = new Date().toISOString();
        const emoji = level === 'INFO' ? '📝' : level === 'WARN' ? '⚠️' : '❌';
        console.log(`  ${emoji} [${timestamp}] ${level}: ${message}`);
    }
    
    logTransaction(accountNumber: string, type: string, amount: number): void {
        this.log('INFO', `Transaction: ${type} ${amount} AED on account ${accountNumber}`);
    }
    
    logError(error: Error): void {
        this.log('ERROR', `Error occurred: ${error.message}`);
    }
}

// ============================================================
// 8. Account Service - ORCHESTRATES other services
// ============================================================
class AccountService {
    constructor(
        private repository: AccountRepository,
        private transactionService: TransactionService,
        private interestCalculator: InterestCalculator,
        private notificationService: NotificationService,
        private auditLogger: AuditLogger
    ) {}
    
    createAccount(accountNumber: string, initialBalance: number): Account {
        this.auditLogger.log('INFO', `Creating account ${accountNumber}`);
        
        const account = new Account(accountNumber, initialBalance);
        this.repository.save(account);
        
        this.notificationService.notifyAll(
            accountNumber,
            `Your account ${accountNumber} has been created successfully!`
        );
        
        return account;
    }
    
    performDeposit(accountNumber: string, amount: number): void {
        const account = this.repository.findByAccountNumber(accountNumber);
        if (!account) {
            throw new Error('Account not found');
        }
        
        this.transactionService.deposit(account, amount);
        this.repository.save(account);
        this.auditLogger.logTransaction(accountNumber, 'DEPOSIT', amount);
        
        this.notificationService.notifyAll(
            accountNumber,
            `Deposit of ${amount} AED successful. New balance: ${account.balance} AED`
        );
    }
    
    applyInterest(accountNumber: string, annualRate: number): void {
        const account = this.repository.findByAccountNumber(accountNumber);
        if (!account) {
            throw new Error('Account not found');
        }
        
        const interest = this.interestCalculator.calculateMonthlyInterest(account.balance, annualRate);
        this.transactionService.deposit(account, interest);
        this.repository.save(account);
        
        this.auditLogger.log('INFO', `Interest applied: ${interest} AED to ${accountNumber}`);
    }
}

// ============================================================
// DEMONSTRATION
// ============================================================

console.log('Creating SRP-compliant banking system:\n');

// Initialize all services
const repository = new AccountRepository();
const transactionService = new TransactionService();
const interestCalculator = new InterestCalculator();
const auditLogger = new AuditLogger();
const statementGenerator = new StatementGenerator();

// Setup notification channels
const notificationService = new NotificationService();
notificationService.addChannel(new EmailNotificationService());
notificationService.addChannel(new SMSNotificationService());

// Create account service with all dependencies
const accountService = new AccountService(
    repository,
    transactionService,
    interestCalculator,
    notificationService,
    auditLogger
);

// Test the system
console.log('1️⃣  Creating Account:');
const account = accountService.createAccount('ACC-12345', 10000);
console.log();

console.log('2️⃣  Performing Deposit:');
accountService.performDeposit('ACC-12345', 5000);
console.log();

console.log('3️⃣  Applying Interest (2.5% annual):');
accountService.applyInterest('ACC-12345', 2.5);
console.log();

console.log('4️⃣  Generating Statement:');
const statement = statementGenerator.generateAccountStatement(
    repository.findByAccountNumber('ACC-12345')!
);
console.log(statement);
console.log();

console.log('5️⃣  Generating Transaction Report:');
const transactions = [
    { type: 'DEPOSIT', amount: 10000, date: new Date('2025-01-01') },
    { type: 'DEPOSIT', amount: 5000, date: new Date('2025-01-15') },
    { type: 'INTEREST', amount: 31.25, date: new Date('2025-01-31') }
];
console.log(statementGenerator.generateTransactionReport(transactions));

console.log('\n' + '='.repeat(70) + '\n');
```

---

## Benefits Comparison

```typescript
console.log('=== SRP Benefits Comparison ===\n');

console.log('WITHOUT SRP (Bad Design):');
console.log('  ❌ AccountManager class has 7 responsibilities');
console.log('  ❌ 200+ lines in one class');
console.log('  ❌ Cannot reuse notification logic separately');
console.log('  ❌ Testing requires mocking everything');
console.log('  ❌ Changes to email logic might break deposits');
console.log('  ❌ Hard to add new notification channels');
console.log('  ❌ Multiple developers cause merge conflicts\n');

console.log('WITH SRP (Good Design):');
console.log('  ✅ 8 focused classes, each with ONE responsibility');
console.log('  ✅ Each class is 20-50 lines (easy to understand)');
console.log('  ✅ Can reuse NotificationService anywhere');
console.log('  ✅ Test each service independently');
console.log('  ✅ Email changes dont affect transaction logic');
console.log('  ✅ Easy to add new channels (WhatsApp, Telegram)');
console.log('  ✅ Different developers work on different classes\n');

console.log('Real-World Scenarios:\n');

console.log('Scenario 1: Add WhatsApp Notifications');
console.log('  Without SRP: Modify 200-line AccountManager class ❌');
console.log('  With SRP: Create WhatsAppNotificationService, add to NotificationService ✅\n');

console.log('Scenario 2: Change Interest Calculation Formula');
console.log('  Without SRP: Risk breaking account operations ❌');
console.log('  With SRP: Only modify InterestCalculator ✅\n');

console.log('Scenario 3: Switch Database from MySQL to MongoDB');
console.log('  Without SRP: Rewrite AccountManager ❌');
console.log('  With SRP: Only modify AccountRepository ✅\n');

console.log('Scenario 4: Unit Testing Notification Logic');
console.log('  Without SRP: Mock database, transactions, interest, etc. ❌');
console.log('  With SRP: Test NotificationService in isolation ✅\n');

console.log('='.repeat(70) + '\n');
```

---

## Interview Questions & Answers

### Q1: What is Single Responsibility Principle?
**Answer**: SRP states that a class should have only ONE reason to change. Each class should have a single, well-defined responsibility. In banking, an `AccountRepository` should only handle data persistence, not send emails or calculate interest.

### Q2: How do you identify SRP violations?
**Answer**: Look for:
- Classes with multiple unrelated methods
- Classes that change for different reasons
- Classes with names like "Manager", "Handler", "Util"
- Methods that do too many things
- High number of dependencies

### Q3: What are the benefits of following SRP?
**Answer**:
1. **Maintainability** - Easier to modify single-purpose classes
2. **Testability** - Simple to write unit tests
3. **Reusability** - Can use classes in different contexts
4. **Reduced coupling** - Classes are independent
5. **Team collaboration** - Developers work on separate classes

### Q4: Give a banking example of SRP violation
**Answer**: A `BankAccount` class that:
- Manages account data ✓
- Calculates interest ✗
- Sends notifications ✗
- Generates statements ✗
- Logs transactions ✗

Should be split into: `Account`, `InterestCalculator`, `NotificationService`, `StatementGenerator`, `AuditLogger`.

### Q5: How does SRP relate to microservices?
**Answer**: SRP applies at service level too. Each microservice should have ONE business responsibility:
- Account Service - Manages accounts
- Transaction Service - Handles transactions
- Notification Service - Sends notifications
- Not: "BankingService" that does everything

---

## Key Takeaways

### ✅ DO:
- Create focused classes with single responsibility
- Use dependency injection for flexibility
- Follow naming conventions that reflect responsibility
- Write small, cohesive classes (50-100 lines max)
- Extract responsibilities into separate classes

### ❌ DON'T:
- Create "God objects" that do everything
- Mix business logic with infrastructure (DB, email)
- Use generic names like "Manager", "Helper", "Util"
- Have methods that do multiple unrelated things
- Fear creating more classes (it's better!)

### Remember:
> "A class should have only ONE reason to change"
> 
> If you need to modify a class for different reasons (add notification channel, change DB, update interest formula), it's violating SRP!

---

## Summary

**Single Responsibility Principle** is the foundation of clean, maintainable code. In banking applications:

- ✅ Separate data (Account) from operations (TransactionService)
- ✅ Keep persistence logic in repositories
- ✅ Extract calculations to dedicated calculators
- ✅ Isolate notification logic
- ✅ Use orchestration services to coordinate

Following SRP makes your banking application:
- **Easier to understand** - Each class has clear purpose
- **Safer to modify** - Changes are isolated
- **Simpler to test** - Test one thing at a time
- **More flexible** - Easy to add features
- **Team-friendly** - Developers work independently

**The key question**: "Does this class have more than one reason to change?"
If YES, split it into multiple classes!
