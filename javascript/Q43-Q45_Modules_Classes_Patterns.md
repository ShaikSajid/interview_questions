# Questions 43-45: Modules, Classes & Advanced OOP

## Question 43: Explain JavaScript Modules - ES6 Modules, CommonJS, and Module Patterns

### Answer:

**Modules** allow you to organize code into reusable, maintainable pieces. JavaScript has evolved from IIFE patterns to native ES6 modules.

### Module Types:
1. **ES6 Modules** - Native JavaScript modules (import/export)
2. **CommonJS** - Node.js module system (require/module.exports)
3. **Module Patterns** - IIFE-based encapsulation
4. **AMD/UMD** - Legacy browser module systems

### Banking Scenario: Module Organization at Emirates NBD

```javascript
console.log('=== JavaScript Modules - Emirates NBD ===\n');

// =============================================================================
// 1. ES6 MODULES (MODERN APPROACH)
// =============================================================================

console.log('1. ES6 MODULES:\n');

// File: account.js (would be separate file)
// export class BankAccount { ... }
// export const MAX_BALANCE = 1000000;
// export default BankingService;

// ES6 Module Syntax Examples:
console.log('ES6 Module Patterns:\n');

console.log('Named Exports:');
console.log('  // account.js');
console.log('  export class BankAccount { }');
console.log('  export const MAX_BALANCE = 1000000;');
console.log('  export function validateAccount() { }\n');

console.log('  // app.js');
console.log('  import { BankAccount, MAX_BALANCE } from "./account.js";');
console.log('  import * as AccountModule from "./account.js";\n');

console.log('Default Exports:');
console.log('  // service.js');
console.log('  export default class BankingService { }\n');

console.log('  // app.js');
console.log('  import BankingService from "./service.js";');
console.log('  import Service from "./service.js"; // Can rename\n');

console.log('Mixed Exports:');
console.log('  export default BankingService;');
console.log('  export { validateAccount, processTransaction };\n');

console.log('  import BankingService, { validateAccount } from "./service.js";\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. COMMONJS MODULES (NODE.JS)
// =============================================================================

console.log('2. COMMONJS MODULES:\n');

console.log('CommonJS Syntax:\n');

console.log('Exporting:');
console.log('  // account.js');
console.log('  class BankAccount { }');
console.log('  const MAX_BALANCE = 1000000;\n');

console.log('  module.exports = BankAccount;');
console.log('  // OR');
console.log('  module.exports = { BankAccount, MAX_BALANCE };\n');

console.log('Importing:');
console.log('  // app.js');
console.log('  const BankAccount = require("./account");');
console.log('  const { BankAccount, MAX_BALANCE } = require("./account");\n');

// Simulated CommonJS implementation
const moduleCache = {};

function require(modulePath) {
    if (moduleCache[modulePath]) {
        console.log(`  Loaded from cache: ${modulePath}`);
        return moduleCache[modulePath].exports;
    }
    
    const module = { exports: {} };
    moduleCache[modulePath] = module;
    
    // Simulate module execution
    console.log(`  Loading module: ${modulePath}`);
    
    return module.exports;
}

console.log('CommonJS Caching:');
require('./account');
require('./account'); // From cache
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. MODULE PATTERN (IIFE)
// =============================================================================

console.log('3. MODULE PATTERN:\n');

// Revealing Module Pattern
const BankingModule = (function() {
    // Private variables and functions
    let accounts = [];
    const API_KEY = 'secret-key';
    
    function validateAccountNumber(accountNumber) {
        return /^ACC-\d{9}$/.test(accountNumber);
    }
    
    function logOperation(operation) {
        console.log(`  [LOG] ${operation}`);
    }
    
    // Public API
    return {
        createAccount(accountNumber, initialBalance) {
            if (!validateAccountNumber(accountNumber)) {
                throw new Error('Invalid account number');
            }
            
            const account = {
                accountNumber,
                balance: initialBalance,
                createdAt: new Date()
            };
            
            accounts.push(account);
            logOperation(`Created account: ${accountNumber}`);
            return account;
        },
        
        getAccount(accountNumber) {
            return accounts.find(a => a.accountNumber === accountNumber);
        },
        
        getAccountCount() {
            return accounts.length;
        },
        
        deposit(accountNumber, amount) {
            const account = this.getAccount(accountNumber);
            if (!account) throw new Error('Account not found');
            
            account.balance += amount;
            logOperation(`Deposited ${amount} to ${accountNumber}`);
            return account.balance;
        }
    };
})();

console.log('Using Module Pattern:\n');

const acc1 = BankingModule.createAccount('ACC-123456789', 50000);
BankingModule.deposit('ACC-123456789', 5000);
console.log(`\nTotal accounts: ${BankingModule.getAccountCount()}`);
console.log(`Account balance: ${acc1.balance}\n`);

console.log('Private members are not accessible:');
console.log(`  BankingModule.accounts: ${BankingModule.accounts}`);
console.log(`  BankingModule.API_KEY: ${BankingModule.API_KEY}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. NAMESPACE PATTERN
// =============================================================================

console.log('4. NAMESPACE PATTERN:\n');

// Global namespace
const EmiratesNBD = EmiratesNBD || {};

// Account namespace
EmiratesNBD.Account = (function() {
    return {
        create(accountNumber, balance) {
            console.log(`  Created account in Account namespace: ${accountNumber}`);
            return { accountNumber, balance };
        },
        
        validate(accountNumber) {
            return /^ACC-\d{9}$/.test(accountNumber);
        }
    };
})();

// Transaction namespace
EmiratesNBD.Transaction = (function() {
    return {
        process(account, amount) {
            console.log(`  Processing transaction: ${amount}`);
            account.balance += amount;
            return account.balance;
        },
        
        validate(amount) {
            return amount > 0 && amount <= 100000;
        }
    };
})();

console.log('Using Namespace Pattern:\n');

const account = EmiratesNBD.Account.create('ACC-987654321', 30000);
EmiratesNBD.Transaction.process(account, 5000);
console.log(`Final balance: ${account.balance}\n`);

console.log('Benefits:');
console.log('  ✓ Avoids global namespace pollution');
console.log('  ✓ Organized code structure');
console.log('  ✓ Easy to understand hierarchy\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. SINGLETON PATTERN
// =============================================================================

console.log('5. SINGLETON PATTERN:\n');

const DatabaseConnection = (function() {
    let instance;
    
    function createInstance() {
        console.log('  Creating database connection...');
        return {
            connectionId: Math.random().toString(36).substr(2, 9),
            connect() {
                console.log(`  Connected: ${this.connectionId}`);
            },
            query(sql) {
                console.log(`  Executing: ${sql}`);
                return [];
            },
            disconnect() {
                console.log(`  Disconnected: ${this.connectionId}`);
            }
        };
    }
    
    return {
        getInstance() {
            if (!instance) {
                instance = createInstance();
            } else {
                console.log('  Returning existing instance');
            }
            return instance;
        }
    };
})();

console.log('First call:');
const db1 = DatabaseConnection.getInstance();
db1.connect();

console.log('\nSecond call:');
const db2 = DatabaseConnection.getInstance();
console.log(`Same instance: ${db1 === db2}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. PRACTICAL BANKING MODULE STRUCTURE
// =============================================================================

console.log('6. PRACTICAL MODULE STRUCTURE:\n');

// Simulating a complete banking application structure
console.log('Recommended folder structure:\n');

console.log('src/');
console.log('├── config/');
console.log('│   └── database.js          - Database configuration');
console.log('├── models/');
console.log('│   ├── Account.js           - Account model');
console.log('│   ├── Transaction.js       - Transaction model');
console.log('│   └── Customer.js          - Customer model');
console.log('├── services/');
console.log('│   ├── AccountService.js    - Account business logic');
console.log('│   ├── TransactionService.js- Transaction processing');
console.log('│   └── NotificationService.js- Notifications');
console.log('├── controllers/');
console.log('│   ├── AccountController.js - Account API endpoints');
console.log('│   └── TransactionController.js');
console.log('├── middleware/');
console.log('│   ├── auth.js             - Authentication');
console.log('│   ├── validation.js       - Input validation');
console.log('│   └── errorHandler.js     - Error handling');
console.log('├── utils/');
console.log('│   ├── logger.js           - Logging utilities');
console.log('│   ├── validator.js        - Validation helpers');
console.log('│   └── crypto.js           - Encryption utilities');
console.log('└── app.js                  - Application entry point\n');

// Example module implementation
class AccountService {
    constructor() {
        this.accounts = new Map();
    }
    
    createAccount(accountNumber, initialBalance) {
        if (this.accounts.has(accountNumber)) {
            throw new Error('Account already exists');
        }
        
        const account = {
            accountNumber,
            balance: initialBalance,
            transactions: [],
            createdAt: new Date()
        };
        
        this.accounts.set(accountNumber, account);
        console.log(`  Service: Created account ${accountNumber}`);
        return account;
    }
    
    getAccount(accountNumber) {
        return this.accounts.get(accountNumber);
    }
    
    deposit(accountNumber, amount) {
        const account = this.getAccount(accountNumber);
        if (!account) throw new Error('Account not found');
        
        account.balance += amount;
        account.transactions.push({
            type: 'DEPOSIT',
            amount,
            timestamp: new Date()
        });
        
        console.log(`  Service: Deposited ${amount} to ${accountNumber}`);
        return account;
    }
}

class AccountController {
    constructor(accountService) {
        this.accountService = accountService;
    }
    
    handleCreateAccount(req) {
        console.log(`  Controller: Handling create account request`);
        
        try {
            const account = this.accountService.createAccount(
                req.accountNumber,
                req.initialBalance
            );
            
            return {
                status: 200,
                data: account
            };
        } catch (error) {
            return {
                status: 400,
                error: error.message
            };
        }
    }
    
    handleDeposit(req) {
        console.log(`  Controller: Handling deposit request`);
        
        try {
            const account = this.accountService.deposit(
                req.accountNumber,
                req.amount
            );
            
            return {
                status: 200,
                data: { balance: account.balance }
            };
        } catch (error) {
            return {
                status: 400,
                error: error.message
            };
        }
    }
}

console.log('Layered Architecture in Action:\n');

const accountService = new AccountService();
const accountController = new AccountController(accountService);

// Create account
const createResponse = accountController.handleCreateAccount({
    accountNumber: 'ACC-111111111',
    initialBalance: 50000
});
console.log(`  Response: Status ${createResponse.status}\n`);

// Deposit
const depositResponse = accountController.handleDeposit({
    accountNumber: 'ACC-111111111',
    amount: 5000
});
console.log(`  Response: Status ${depositResponse.status}, Balance: ${depositResponse.data.balance}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// MODULES SUMMARY
// =============================================================================

console.log('MODULES SUMMARY:\n');

console.log('ES6 Modules:');
console.log('  ✓ Native JavaScript modules');
console.log('  ✓ Static imports (tree-shakable)');
console.log('  ✓ Strict mode by default');
console.log('  ✓ Asynchronous loading');
console.log('  ✓ Preferred for modern applications\n');

console.log('CommonJS:');
console.log('  ✓ Node.js default');
console.log('  ✓ Synchronous loading');
console.log('  ✓ Dynamic require()');
console.log('  ✓ Module caching\n');

console.log('Module Patterns:');
console.log('  ✓ IIFE for encapsulation');
console.log('  ✓ Revealing module pattern');
console.log('  ✓ Namespace pattern');
console.log('  ✓ Singleton pattern\n');

console.log('Best Practices:');
console.log('  ✓ One module per file');
console.log('  ✓ Clear, descriptive names');
console.log('  ✓ Export only what\'s necessary');
console.log('  ✓ Use barrel exports (index.js)');
console.log('  ✓ Avoid circular dependencies');
console.log('  ✓ Keep modules focused (Single Responsibility)');

console.log();
```

---

## Question 44: Explain ES6 Classes, inheritance, and differences from prototypal inheritance

### Answer:

**ES6 Classes** provide syntactic sugar over JavaScript's prototypal inheritance, making OOP patterns more familiar to developers from other languages.

### Key Features:
1. **Class Declaration** - class keyword
2. **Constructor** - Initialize instances
3. **Methods** - Instance and static methods
4. **Inheritance** - extends keyword
5. **Super** - Call parent class
6. **Private Fields** - # prefix (ES2022)

### Banking Scenario: Class Hierarchy at Emirates NBD

```javascript
console.log('=== ES6 Classes & Inheritance - Emirates NBD ===\n');

// =============================================================================
// 1. BASIC CLASS SYNTAX
// =============================================================================

console.log('1. BASIC CLASS SYNTAX:\n');

class BankAccount {
    // Private fields (ES2022)
    #balance;
    #pin;
    
    // Constructor
    constructor(accountNumber, initialBalance, pin) {
        this.accountNumber = accountNumber;
        this.#balance = initialBalance;
        this.#pin = pin;
        this.createdAt = new Date();
        this.transactions = [];
        
        console.log(`  Created account: ${accountNumber}`);
    }
    
    // Public method
    deposit(amount, pin) {
        if (!this.#verifyPin(pin)) {
            throw new Error('Invalid PIN');
        }
        
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        
        this.#balance += amount;
        this.#recordTransaction('DEPOSIT', amount);
        console.log(`  Deposited: ${amount}`);
        return this.#balance;
    }
    
    // Public method
    withdraw(amount, pin) {
        if (!this.#verifyPin(pin)) {
            throw new Error('Invalid PIN');
        }
        
        if (amount > this.#balance) {
            throw new Error('Insufficient funds');
        }
        
        this.#balance -= amount;
        this.#recordTransaction('WITHDRAWAL', amount);
        console.log(`  Withdrew: ${amount}`);
        return this.#balance;
    }
    
    // Public getter
    get balance() {
        return this.#balance;
    }
    
    // Private method
    #verifyPin(pin) {
        return pin === this.#pin;
    }
    
    // Private method
    #recordTransaction(type, amount) {
        this.transactions.push({
            type,
            amount,
            balance: this.#balance,
            timestamp: new Date()
        });
    }
    
    // Static method
    static validateAccountNumber(accountNumber) {
        return /^ACC-\d{9}$/.test(accountNumber);
    }
}

const account = new BankAccount('ACC-123456789', 50000, '1234');
account.deposit(5000, '1234');
console.log(`Balance: ${account.balance}\n`);

// Private fields are not accessible
console.log('Private field access:');
console.log(`  account.#balance: undefined (private)`);
console.log(`  account.balance: ${account.balance} (getter)\n`);

// Static method
console.log('Static method:');
console.log(`  Valid: ${BankAccount.validateAccountNumber('ACC-123456789')}`);
console.log(`  Invalid: ${BankAccount.validateAccountNumber('INVALID')}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. INHERITANCE WITH EXTENDS
// =============================================================================

console.log('2. INHERITANCE:\n');

// Parent class
class Account {
    constructor(accountNumber, balance) {
        this.accountNumber = accountNumber;
        this.balance = balance;
        this.transactions = [];
    }
    
    deposit(amount) {
        this.balance += amount;
        this.recordTransaction('DEPOSIT', amount);
        return this.balance;
    }
    
    withdraw(amount) {
        if (amount > this.balance) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
        this.recordTransaction('WITHDRAWAL', amount);
        return this.balance;
    }
    
    recordTransaction(type, amount) {
        this.transactions.push({ type, amount, timestamp: new Date() });
    }
    
    getBalance() {
        return this.balance;
    }
}

// Child class - Savings Account
class SavingsAccount extends Account {
    constructor(accountNumber, balance, interestRate) {
        super(accountNumber, balance); // Call parent constructor
        this.interestRate = interestRate;
        this.minimumBalance = 1000;
    }
    
    // Override parent method
    withdraw(amount) {
        if (this.balance - amount < this.minimumBalance) {
            throw new Error(`Balance cannot go below ${this.minimumBalance}`);
        }
        return super.withdraw(amount); // Call parent method
    }
    
    // New method
    calculateInterest() {
        const interest = this.balance * this.interestRate;
        console.log(`  Interest calculated: ${interest.toFixed(2)}`);
        return interest;
    }
    
    // New method
    applyInterest() {
        const interest = this.calculateInterest();
        this.deposit(interest);
        console.log(`  Interest applied: ${interest.toFixed(2)}`);
        return this.balance;
    }
}

// Child class - Current Account
class CurrentAccount extends Account {
    constructor(accountNumber, balance, overdraftLimit) {
        super(accountNumber, balance);
        this.overdraftLimit = overdraftLimit;
    }
    
    // Override parent method
    withdraw(amount) {
        if (amount > this.balance + this.overdraftLimit) {
            throw new Error('Exceeds overdraft limit');
        }
        this.balance -= amount;
        this.recordTransaction('WITHDRAWAL', amount);
        return this.balance;
    }
    
    // New method
    getAvailableBalance() {
        return this.balance + this.overdraftLimit;
    }
}

console.log('Savings Account:');
const savings = new SavingsAccount('ACC-111111111', 50000, 0.035);
savings.deposit(10000);
console.log(`  Balance: ${savings.getBalance()}`);
savings.calculateInterest();
console.log();

console.log('Current Account:');
const current = new CurrentAccount('ACC-222222222', 30000, 10000);
console.log(`  Balance: ${current.getBalance()}`);
console.log(`  Available: ${current.getAvailableBalance()}`);
current.withdraw(35000); // Uses overdraft
console.log(`  After withdrawal: ${current.getBalance()}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. CLASS VS PROTOTYPAL INHERITANCE
// =============================================================================

console.log('3. CLASS VS PROTOTYPAL INHERITANCE:\n');

// Prototypal inheritance (ES5)
console.log('Prototypal Inheritance (ES5):\n');

function BankAccountES5(accountNumber, balance) {
    this.accountNumber = accountNumber;
    this.balance = balance;
}

BankAccountES5.prototype.deposit = function(amount) {
    this.balance += amount;
    console.log(`  ES5 deposit: ${amount}`);
    return this.balance;
};

BankAccountES5.prototype.getBalance = function() {
    return this.balance;
};

const es5Account = new BankAccountES5('ACC-333333333', 40000);
es5Account.deposit(5000);
console.log(`  Balance: ${es5Account.getBalance()}\n`);

// Class inheritance (ES6)
console.log('Class Inheritance (ES6):\n');

class BankAccountES6 {
    constructor(accountNumber, balance) {
        this.accountNumber = accountNumber;
        this.balance = balance;
    }
    
    deposit(amount) {
        this.balance += amount;
        console.log(`  ES6 deposit: ${amount}`);
        return this.balance;
    }
    
    getBalance() {
        return this.balance;
    }
}

const es6Account = new BankAccountES6('ACC-444444444', 40000);
es6Account.deposit(5000);
console.log(`  Balance: ${es6Account.getBalance()}\n`);

console.log('Key Differences:\n');
console.log('ES5 Prototypal:');
console.log('  • Constructor function');
console.log('  • Methods on prototype');
console.log('  • Manual inheritance setup');
console.log('  • No private fields');
console.log('  • Can be called without new\n');

console.log('ES6 Classes:');
console.log('  • class keyword');
console.log('  • Methods in class body');
console.log('  • extends for inheritance');
console.log('  • Private fields with #');
console.log('  • Must use new keyword\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. GETTERS AND SETTERS
// =============================================================================

console.log('4. GETTERS AND SETTERS:\n');

class SecureAccount {
    #balance;
    
    constructor(accountNumber, initialBalance) {
        this.accountNumber = accountNumber;
        this.#balance = initialBalance;
    }
    
    // Getter
    get balance() {
        console.log('  Getter called');
        return this.#balance;
    }
    
    // Setter
    set balance(value) {
        console.log('  Setter called');
        if (value < 0) {
            throw new Error('Balance cannot be negative');
        }
        this.#balance = value;
    }
    
    // Computed property
    get formattedBalance() {
        return `AED ${this.#balance.toLocaleString()}`;
    }
}

const secureAccount = new SecureAccount('ACC-555555555', 50000);

console.log('Using getter:');
console.log(`  ${secureAccount.balance}\n`);

console.log('Using setter:');
secureAccount.balance = 60000;
console.log(`  New balance: ${secureAccount.balance}\n`);

console.log('Computed property:');
console.log(`  ${secureAccount.formattedBalance}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. STATIC METHODS AND PROPERTIES
// =============================================================================

console.log('5. STATIC METHODS AND PROPERTIES:\n');

class BankingUtils {
    static MIN_BALANCE = 1000;
    static MAX_TRANSFER = 100000;
    
    static validateAmount(amount) {
        return amount > 0 && amount <= this.MAX_TRANSFER;
    }
    
    static calculateInterest(balance, rate, years) {
        return balance * Math.pow(1 + rate, years) - balance;
    }
    
    static formatCurrency(amount) {
        return `AED ${amount.toLocaleString('en-AE', {
            minimumFractionDigits: 2,
            maximumFractionDigits: 2
        })}`;
    }
}

console.log('Static properties:');
console.log(`  MIN_BALANCE: ${BankingUtils.MIN_BALANCE}`);
console.log(`  MAX_TRANSFER: ${BankingUtils.MAX_TRANSFER}\n`);

console.log('Static methods:');
console.log(`  Valid amount: ${BankingUtils.validateAmount(50000)}`);
console.log(`  Invalid amount: ${BankingUtils.validateAmount(200000)}`);

const interest = BankingUtils.calculateInterest(100000, 0.035, 5);
console.log(`  Interest: ${BankingUtils.formatCurrency(interest)}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. PRACTICAL BANKING CLASS HIERARCHY
// =============================================================================

console.log('6. PRACTICAL CLASS HIERARCHY:\n');

class BaseAccount {
    constructor(accountNumber, accountType) {
        if (new.target === BaseAccount) {
            throw new Error('BaseAccount is abstract and cannot be instantiated');
        }
        
        this.accountNumber = accountNumber;
        this.accountType = accountType;
        this.balance = 0;
        this.transactions = [];
        this.status = 'ACTIVE';
    }
    
    deposit(amount) {
        if (this.status !== 'ACTIVE') {
            throw new Error('Account is not active');
        }
        
        this.balance += amount;
        this.addTransaction('DEPOSIT', amount);
        return this.balance;
    }
    
    withdraw(amount) {
        throw new Error('withdraw() must be implemented by subclass');
    }
    
    addTransaction(type, amount) {
        this.transactions.push({
            type,
            amount,
            balance: this.balance,
            timestamp: new Date()
        });
    }
    
    getStatement() {
        return {
            accountNumber: this.accountNumber,
            accountType: this.accountType,
            balance: this.balance,
            status: this.status,
            transactions: this.transactions
        };
    }
}

class SavingsAccountPro extends BaseAccount {
    constructor(accountNumber, interestRate) {
        super(accountNumber, 'SAVINGS');
        this.interestRate = interestRate;
        this.minimumBalance = 1000;
    }
    
    withdraw(amount) {
        if (this.balance - amount < this.minimumBalance) {
            throw new Error('Minimum balance requirement not met');
        }
        
        this.balance -= amount;
        this.addTransaction('WITHDRAWAL', amount);
        return this.balance;
    }
    
    creditMonthlyInterest() {
        const interest = this.balance * (this.interestRate / 12);
        this.deposit(interest);
        console.log(`  Monthly interest credited: ${interest.toFixed(2)}`);
        return this.balance;
    }
}

class CurrentAccountPro extends BaseAccount {
    constructor(accountNumber, overdraftLimit) {
        super(accountNumber, 'CURRENT');
        this.overdraftLimit = overdraftLimit;
        this.monthlyFee = 50;
    }
    
    withdraw(amount) {
        if (amount > this.balance + this.overdraftLimit) {
            throw new Error('Insufficient funds including overdraft');
        }
        
        this.balance -= amount;
        this.addTransaction('WITHDRAWAL', amount);
        return this.balance;
    }
    
    deductMonthlyFee() {
        this.balance -= this.monthlyFee;
        this.addTransaction('FEE', this.monthlyFee);
        console.log(`  Monthly fee deducted: ${this.monthlyFee}`);
        return this.balance;
    }
}

console.log('Testing class hierarchy:\n');

const savingsPro = new SavingsAccountPro('ACC-666666666', 0.035);
savingsPro.deposit(50000);
console.log(`Savings balance: ${savingsPro.balance}`);
savingsPro.creditMonthlyInterest();
console.log();

const currentPro = new CurrentAccountPro('ACC-777777777', 10000);
currentPro.deposit(30000);
console.log(`Current balance: ${currentPro.balance}`);
currentPro.withdraw(35000); // Uses overdraft
console.log(`After overdraft: ${currentPro.balance}`);
currentPro.deductMonthlyFee();
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// ES6 CLASSES SUMMARY
// =============================================================================

console.log('ES6 CLASSES SUMMARY:\n');

console.log('Class Features:');
console.log('  ✓ constructor() - Initialize instances');
console.log('  ✓ Methods - Instance methods');
console.log('  ✓ static - Class methods/properties');
console.log('  ✓ get/set - Computed properties');
console.log('  ✓ # prefix - Private fields (ES2022)');
console.log('  ✓ extends - Inheritance');
console.log('  ✓ super - Parent class access\n');

console.log('Advantages over Prototypal:');
console.log('  ✓ Cleaner syntax');
console.log('  ✓ Better for OOP patterns');
console.log('  ✓ Private fields support');
console.log('  ✓ Familiar to developers from other languages');
console.log('  ✓ Better tooling support\n');

console.log('Best Practices:');
console.log('  ✓ Use classes for OOP designs');
console.log('  ✓ Keep classes focused (Single Responsibility)');
console.log('  ✓ Use private fields for encapsulation');
console.log('  ✓ Prefer composition over deep inheritance');
console.log('  ✓ Use static methods for utilities');
console.log('  ✓ Document public API clearly');

console.log();
```

---

## Question 45: What are Design Patterns in JavaScript? Explain common patterns used in applications.

### Answer:

**Design Patterns** are reusable solutions to common programming problems. They provide tested, proven development paradigms that can speed up development and make code more maintainable.

### Common Patterns:
1. **Creational** - Object creation (Singleton, Factory, Builder)
2. **Structural** - Object composition (Module, Decorator, Facade)
3. **Behavioral** - Object interaction (Observer, Strategy, Command)

### Banking Scenario: Design Patterns at Emirates NBD

```javascript
console.log('=== Design Patterns - Emirates NBD ===\n');

// =============================================================================
// 1. FACTORY PATTERN
// =============================================================================

console.log('1. FACTORY PATTERN:\n');

class AccountFactory {
    static createAccount(type, accountNumber, balance, ...args) {
        switch (type) {
            case 'SAVINGS':
                return new SavingsAccountFactory(accountNumber, balance, args[0]);
            case 'CURRENT':
                return new CurrentAccountFactory(accountNumber, balance, args[0]);
            case 'FIXED':
                return new FixedDepositAccount(accountNumber, balance, args[0], args[1]);
            default:
                throw new Error('Unknown account type');
        }
    }
}

class SavingsAccountFactory {
    constructor(accountNumber, balance, interestRate) {
        this.accountNumber = accountNumber;
        this.balance = balance;
        this.type = 'SAVINGS';
        this.interestRate = interestRate;
    }
    
    getInfo() {
        return `${this.type} - ${this.accountNumber}: ${this.balance}`;
    }
}

class CurrentAccountFactory {
    constructor(accountNumber, balance, overdraftLimit) {
        this.accountNumber = accountNumber;
        this.balance = balance;
        this.type = 'CURRENT';
        this.overdraftLimit = overdraftLimit;
    }
    
    getInfo() {
        return `${this.type} - ${this.accountNumber}: ${this.balance}`;
    }
}

class FixedDepositAccount {
    constructor(accountNumber, balance, interestRate, term) {
        this.accountNumber = accountNumber;
        this.balance = balance;
        this.type = 'FIXED';
        this.interestRate = interestRate;
        this.term = term;
    }
    
    getInfo() {
        return `${this.type} - ${this.accountNumber}: ${this.balance}`;
    }
}

console.log('Creating accounts with factory:\n');

const savings = AccountFactory.createAccount('SAVINGS', 'ACC-001', 50000, 0.035);
const current = AccountFactory.createAccount('CURRENT', 'ACC-002', 30000, 10000);
const fixed = AccountFactory.createAccount('FIXED', 'ACC-003', 100000, 0.05, 12);

console.log(`  ${savings.getInfo()}`);
console.log(`  ${current.getInfo()}`);
console.log(`  ${fixed.getInfo()}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. OBSERVER PATTERN (PUB/SUB)
// =============================================================================

console.log('2. OBSERVER PATTERN:\n');

class TransactionObserver {
    constructor() {
        this.observers = [];
    }
    
    subscribe(observer) {
        this.observers.push(observer);
        console.log(`  Subscribed: ${observer.name}`);
    }
    
    unsubscribe(observer) {
        this.observers = this.observers.filter(obs => obs !== observer);
        console.log(`  Unsubscribed: ${observer.name}`);
    }
    
    notify(transaction) {
        console.log(`\n  Notifying ${this.observers.length} observers...`);
        this.observers.forEach(observer => observer.update(transaction));
    }
}

class EmailNotifier {
    constructor() {
        this.name = 'EmailNotifier';
    }
    
    update(transaction) {
        console.log(`  📧 Email: Transaction ${transaction.id} - ${transaction.amount} AED`);
    }
}

class SMSNotifier {
    constructor() {
        this.name = 'SMSNotifier';
    }
    
    update(transaction) {
        console.log(`  📱 SMS: Transaction ${transaction.id} - ${transaction.amount} AED`);
    }
}

class LoggerService {
    constructor() {
        this.name = 'LoggerService';
    }
    
    update(transaction) {
        console.log(`  📝 Log: [${new Date().toISOString()}] ${transaction.id}`);
    }
}

const transactionSubject = new TransactionObserver();

const emailNotifier = new EmailNotifier();
const smsNotifier = new SMSNotifier();
const logger = new LoggerService();

console.log('Setting up observers:\n');
transactionSubject.subscribe(emailNotifier);
transactionSubject.subscribe(smsNotifier);
transactionSubject.subscribe(logger);

console.log('\nTriggering transaction:');
transactionSubject.notify({ id: 'TXN-001', amount: 5000 });

console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. STRATEGY PATTERN
// =============================================================================

console.log('3. STRATEGY PATTERN:\n');

// Interest calculation strategies
class StandardInterestStrategy {
    calculate(balance) {
        return balance * 0.03;
    }
    
    getName() {
        return 'Standard (3%)';
    }
}

class PremiumInterestStrategy {
    calculate(balance) {
        if (balance > 100000) {
            return balance * 0.05;
        }
        return balance * 0.035;
    }
    
    getName() {
        return 'Premium (3.5%-5%)';
    }
}

class SeniorCitizenStrategy {
    calculate(balance) {
        return balance * 0.06;
    }
    
    getName() {
        return 'Senior Citizen (6%)';
    }
}

class InterestCalculator {
    constructor(strategy) {
        this.strategy = strategy;
    }
    
    setStrategy(strategy) {
        this.strategy = strategy;
    }
    
    calculate(balance) {
        const interest = this.strategy.calculate(balance);
        console.log(`  ${this.strategy.getName()}: ${interest.toFixed(2)} AED`);
        return interest;
    }
}

console.log('Calculating interest with different strategies:\n');

const calculator = new InterestCalculator(new StandardInterestStrategy());
calculator.calculate(50000);

calculator.setStrategy(new PremiumInterestStrategy());
calculator.calculate(150000);

calculator.setStrategy(new SeniorCitizenStrategy());
calculator.calculate(50000);

console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. DECORATOR PATTERN
// =============================================================================

console.log('4. DECORATOR PATTERN:\n');

class BasicAccount {
    constructor(accountNumber) {
        this.accountNumber = accountNumber;
        this.features = ['Basic Banking'];
    }
    
    getFeatures() {
        return this.features;
    }
    
    getMonthlyFee() {
        return 0;
    }
}

// Decorators
class OnlineBankingDecorator {
    constructor(account) {
        this.account = account;
    }
    
    getFeatures() {
        return [...this.account.getFeatures(), 'Online Banking'];
    }
    
    getMonthlyFee() {
        return this.account.getMonthlyFee() + 10;
    }
}

class DebitCardDecorator {
    constructor(account) {
        this.account = account;
    }
    
    getFeatures() {
        return [...this.account.getFeatures(), 'Debit Card'];
    }
    
    getMonthlyFee() {
        return this.account.getMonthlyFee() + 5;
    }
}

class SMSAlertsDecorator {
    constructor(account) {
        this.account = account;
    }
    
    getFeatures() {
        return [...this.account.getFeatures(), 'SMS Alerts'];
    }
    
    getMonthlyFee() {
        return this.account.getMonthlyFee() + 3;
    }
}

let account1 = new BasicAccount('ACC-888888888');
console.log('Basic Account:');
console.log(`  Features: ${account1.getFeatures().join(', ')}`);
console.log(`  Monthly fee: ${account1.getMonthlyFee()} AED\n`);

account1 = new OnlineBankingDecorator(account1);
account1 = new DebitCardDecorator(account1);
account1 = new SMSAlertsDecorator(account1);

console.log('Enhanced Account:');
console.log(`  Features: ${account1.getFeatures().join(', ')}`);
console.log(`  Monthly fee: ${account1.getMonthlyFee()} AED\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. COMMAND PATTERN
// =============================================================================

console.log('5. COMMAND PATTERN:\n');

class Transaction {
    constructor(account) {
        this.account = account;
    }
    
    execute() {
        throw new Error('execute() must be implemented');
    }
    
    undo() {
        throw new Error('undo() must be implemented');
    }
}

class DepositCommand extends Transaction {
    constructor(account, amount) {
        super(account);
        this.amount = amount;
    }
    
    execute() {
        this.account.balance += this.amount;
        console.log(`  Executed: Deposit ${this.amount}`);
    }
    
    undo() {
        this.account.balance -= this.amount;
        console.log(`  Undone: Deposit ${this.amount}`);
    }
}

class WithdrawCommand extends Transaction {
    constructor(account, amount) {
        super(account);
        this.amount = amount;
    }
    
    execute() {
        this.account.balance -= this.amount;
        console.log(`  Executed: Withdraw ${this.amount}`);
    }
    
    undo() {
        this.account.balance += this.amount;
        console.log(`  Undone: Withdraw ${this.amount}`);
    }
}

class TransactionManager {
    constructor() {
        this.history = [];
    }
    
    execute(command) {
        command.execute();
        this.history.push(command);
    }
    
    undo() {
        const command = this.history.pop();
        if (command) {
            command.undo();
        }
    }
}

const commandAccount = { balance: 50000 };
const manager = new TransactionManager();

console.log(`Initial balance: ${commandAccount.balance}\n`);

manager.execute(new DepositCommand(commandAccount, 5000));
console.log(`Balance: ${commandAccount.balance}\n`);

manager.execute(new WithdrawCommand(commandAccount, 2000));
console.log(`Balance: ${commandAccount.balance}\n`);

manager.undo();
console.log(`Balance after undo: ${commandAccount.balance}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// DESIGN PATTERNS SUMMARY
// =============================================================================

console.log('DESIGN PATTERNS SUMMARY:\n');

console.log('Creational Patterns:');
console.log('  • Singleton - Single instance');
console.log('  • Factory - Object creation');
console.log('  • Builder - Complex object construction');
console.log('  • Prototype - Clone existing objects\n');

console.log('Structural Patterns:');
console.log('  • Module - Encapsulation');
console.log('  • Decorator - Add features dynamically');
console.log('  • Facade - Simplified interface');
console.log('  • Adapter - Interface compatibility\n');

console.log('Behavioral Patterns:');
console.log('  • Observer - Event notification');
console.log('  • Strategy - Interchangeable algorithms');
console.log('  • Command - Encapsulate actions');
console.log('  • Iterator - Sequential access\n');

console.log('When to Use:');
console.log('  ✓ Singleton - Database connections, config');
console.log('  ✓ Factory - Multiple object types');
console.log('  ✓ Observer - Event-driven systems');
console.log('  ✓ Strategy - Multiple algorithms');
console.log('  ✓ Decorator - Add features flexibly');
console.log('  ✓ Command - Undo/redo functionality');

console.log();
```

### Key Takeaways:

**Modules**:
- ES6 modules (import/export) for modern apps
- CommonJS (require) for Node.js
- Module patterns for encapsulation
- Keep modules focused and independent

**ES6 Classes**:
- Syntactic sugar over prototypal inheritance
- Private fields with # prefix
- extends for inheritance, super for parent access
- Static methods for utilities
- Better for OOP patterns

**Design Patterns**:
- Reusable solutions to common problems
- Factory for object creation
- Observer for event systems
- Strategy for interchangeable algorithms
- Decorator for adding features
- Command for undo/redo

---

**End of Questions 43-45**
