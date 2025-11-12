# Questions 31-33: Regular Expressions, Symbols & Reflect API

## Question 31: How do you use Regular Expressions in JavaScript? Explain patterns, flags, and methods.

### Answer:

**Regular Expressions (RegEx)** are patterns used to match character combinations in strings. They're powerful for validation, searching, and text manipulation.

### RegEx Components:
- **Patterns**: Character combinations and special sequences
- **Flags**: Modifiers (g, i, m, s, u, y)
- **Methods**: test(), exec(), match(), replace(), search(), split()

### Banking Scenario: Data Validation at Emirates NBD

```javascript
console.log('=== Regular Expressions Demo - Emirates NBD ===\n');

// =============================================================================
// 1. BASIC PATTERNS & SYNTAX
// =============================================================================

console.log('1. BASIC REGEX PATTERNS:\n');

class BankingPatterns {
    // Account number: ACC-XXXXXXXXX (9 digits)
    static ACCOUNT_NUMBER = /^ACC-\d{9}$/;
    
    // IBAN: AE + 2 digits + 16 digits
    static IBAN = /^AE\d{18}$/;
    
    // Credit card: 16 digits with optional spaces/dashes
    static CREDIT_CARD = /^(\d{4}[-\s]?){3}\d{4}$/;
    
    // Emirates ID: XXX-XXXX-XXXXXXX-X
    static EMIRATES_ID = /^\d{3}-\d{4}-\d{7}-\d$/;
    
    // Mobile: +971-XX-XXX-XXXX or 05XXXXXXXX
    static MOBILE_UAE = /^(\+971-?|0)?5[0-9]{1}[-\s]?\d{3}[-\s]?\d{4}$/;
    
    // Email
    static EMAIL = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
    
    // Amount: Optional currency symbol, digits with optional decimals
    static AMOUNT = /^[A-Z]{3}?\s?\d{1,3}(,?\d{3})*(\.\d{2})?$/;
    
    // Date: DD/MM/YYYY or DD-MM-YYYY
    static DATE = /^(0[1-9]|[12][0-9]|3[01])[\/\-](0[1-9]|1[0-2])[\/\-]\d{4}$/;
    
    // Strong password: 8+ chars, uppercase, lowercase, digit, special char
    static STRONG_PASSWORD = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/;
    
    // Swift/BIC code: 8 or 11 characters
    static SWIFT_BIC = /^[A-Z]{6}[A-Z0-9]{2}([A-Z0-9]{3})?$/;
}

// Demonstrate pattern matching
console.log('Pattern Matching Examples:\n');

const testCases = [
    { pattern: BankingPatterns.ACCOUNT_NUMBER, value: 'ACC-123456789', label: 'Account Number' },
    { pattern: BankingPatterns.IBAN, value: 'AE070331234567890123456', label: 'IBAN' },
    { pattern: BankingPatterns.CREDIT_CARD, value: '4111-1111-1111-1111', label: 'Credit Card' },
    { pattern: BankingPatterns.EMIRATES_ID, value: '784-1988-1234567-1', label: 'Emirates ID' },
    { pattern: BankingPatterns.MOBILE_UAE, value: '+971-50-123-4567', label: 'UAE Mobile' },
    { pattern: BankingPatterns.EMAIL, value: 'customer@emiratesnbd.com', label: 'Email' },
    { pattern: BankingPatterns.AMOUNT, value: 'AED 50,000.00', label: 'Amount' },
    { pattern: BankingPatterns.DATE, value: '15/03/2024', label: 'Date' },
    { pattern: BankingPatterns.SWIFT_BIC, value: 'EBILAEAD', label: 'Swift/BIC' }
];

testCases.forEach(({ pattern, value, label }) => {
    const isValid = pattern.test(value);
    console.log(`${label}:`);
    console.log(`  Value: ${value}`);
    console.log(`  Valid: ${isValid ? '✅' : '❌'}\n`);
});

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. REGEX FLAGS
// =============================================================================

console.log('2. REGEX FLAGS:\n');

class RegexFlags {
    static demonstrateFlags() {
        const text = 'ACC-123456789, acc-987654321, ACC-555555555';
        
        // g - Global (find all matches)
        console.log('g flag (global):');
        const patternG = /acc-\d{9}/gi;
        const matchesG = text.match(patternG);
        console.log(`  Pattern: ${patternG}`);
        console.log(`  Matches: ${matchesG}\n`);
        
        // i - Case insensitive
        console.log('i flag (case insensitive):');
        const patternI = /acc-\d{9}/i;
        const matchI = text.match(patternI);
        console.log(`  Pattern: ${patternI}`);
        console.log(`  First match: ${matchI ? matchI[0] : 'none'}\n`);
        
        // m - Multiline
        console.log('m flag (multiline):');
        const multilineText = `Account: ACC-123456789
Another: ACC-987654321
Last: ACC-555555555`;
        const patternM = /^Account: (ACC-\d{9})$/m;
        const matchM = multilineText.match(patternM);
        console.log(`  Pattern: ${patternM}`);
        console.log(`  Match: ${matchM ? matchM[1] : 'none'}\n`);
        
        // s - Dotall (. matches newlines)
        console.log('s flag (dotall):');
        const textS = 'Transaction\nAmount: 5000';
        const patternS = /Transaction.Amount/s;
        console.log(`  Pattern: ${patternS}`);
        console.log(`  Matches: ${patternS.test(textS)}\n`);
        
        // u - Unicode
        console.log('u flag (unicode):');
        const arabicText = 'الحساب: ACC-123456789';
        const patternU = /\p{Script=Arabic}+/u;
        console.log(`  Pattern: ${patternU}`);
        console.log(`  Matches: ${patternU.test(arabicText)}\n`);
        
        // y - Sticky (match from lastIndex)
        console.log('y flag (sticky):');
        const patternY = /\d+/y;
        const textY = '123 456 789';
        patternY.lastIndex = 0;
        console.log(`  Pattern: ${patternY}`);
        console.log(`  Match at 0: ${patternY.exec(textY)}`);
        patternY.lastIndex = 4;
        console.log(`  Match at 4: ${patternY.exec(textY)}\n`);
    }
}

RegexFlags.demonstrateFlags();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. REGEX METHODS
// =============================================================================

console.log('3. REGEX METHODS:\n');

class RegexMethods {
    // test() - Returns boolean
    static testMethod() {
        console.log('test() method - Returns boolean:\n');
        
        const pattern = /^ACC-\d{9}$/;
        const validAccount = 'ACC-123456789';
        const invalidAccount = 'ACC-12345';
        
        console.log(`  ${validAccount} → ${pattern.test(validAccount)}`);
        console.log(`  ${invalidAccount} → ${pattern.test(invalidAccount)}\n`);
    }
    
    // exec() - Returns match details
    static execMethod() {
        console.log('exec() method - Returns match details:\n');
        
        const pattern = /^(ACC)-(\d{3})(\d{3})(\d{3})$/;
        const account = 'ACC-123456789';
        const match = pattern.exec(account);
        
        if (match) {
            console.log('  Full match:', match[0]);
            console.log('  Prefix:', match[1]);
            console.log('  Part 1:', match[2]);
            console.log('  Part 2:', match[3]);
            console.log('  Part 3:', match[4]);
            console.log('  Index:', match.index);
            console.log('  Input:', match.input);
        }
        console.log();
    }
    
    // match() - Returns array of matches
    static matchMethod() {
        console.log('match() method - Returns array of matches:\n');
        
        const text = 'Transfers: ACC-111111111 to ACC-222222222 amount: AED 5000';
        const pattern = /ACC-\d{9}/g;
        const matches = text.match(pattern);
        
        console.log('  Text:', text);
        console.log('  Matches:', matches);
        console.log();
    }
    
    // matchAll() - Returns iterator of matches with details
    static matchAllMethod() {
        console.log('matchAll() method - Returns iterator with details:\n');
        
        const text = 'Accounts: ACC-111111111, ACC-222222222, ACC-333333333';
        const pattern = /ACC-(\d{3})(\d{3})(\d{3})/g;
        const matches = [...text.matchAll(pattern)];
        
        matches.forEach((match, i) => {
            console.log(`  Match ${i + 1}:`, match[0]);
            console.log(`    Groups: ${match[1]}-${match[2]}-${match[3]}`);
        });
        console.log();
    }
    
    // search() - Returns index of first match
    static searchMethod() {
        console.log('search() method - Returns index of first match:\n');
        
        const text = 'Customer account: ACC-123456789';
        const pattern = /ACC-\d{9}/;
        const index = text.search(pattern);
        
        console.log('  Text:', text);
        console.log('  First match at index:', index);
        console.log('  Match:', text.substring(index, index + 13));
        console.log();
    }
    
    // replace() - Replaces matches
    static replaceMethod() {
        console.log('replace() method - Replaces matches:\n');
        
        const text = 'Account ACC-123456789 has balance of AED 50000';
        
        // Simple replacement
        const masked1 = text.replace(/\d/g, '*');
        console.log('  Mask all digits:', masked1);
        
        // With function
        const masked2 = text.replace(/ACC-(\d{3})(\d{3})(\d{3})/, (match, p1, p2, p3) => {
            return `ACC-${p1}***${p3}`;
        });
        console.log('  Mask middle digits:', masked2);
        
        // Named capture groups
        const pattern = /(?<currency>[A-Z]{3})\s(?<amount>\d+)/;
        const formatted = text.replace(pattern, '$<currency> $<amount>.00');
        console.log('  Format amount:', formatted);
        console.log();
    }
    
    // split() - Splits string by pattern
    static splitMethod() {
        console.log('split() method - Splits string by pattern:\n');
        
        const text = 'ACC-123456789,ACC-987654321;ACC-555555555';
        const accounts = text.split(/[,;]/);
        
        console.log('  Text:', text);
        console.log('  Split accounts:', accounts);
        console.log();
    }
}

RegexMethods.testMethod();
RegexMethods.execMethod();
RegexMethods.matchMethod();
RegexMethods.matchAllMethod();
RegexMethods.searchMethod();
RegexMethods.replaceMethod();
RegexMethods.splitMethod();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. PRACTICAL BANKING VALIDATORS
// =============================================================================

console.log('4. PRACTICAL BANKING VALIDATORS:\n');

class BankingValidator {
    // Validate account number
    static validateAccountNumber(accountNumber) {
        const pattern = /^ACC-\d{9}$/;
        const isValid = pattern.test(accountNumber);
        
        console.log(`Account Number Validation:`);
        console.log(`  Input: ${accountNumber}`);
        console.log(`  Valid: ${isValid ? '✅' : '❌'}\n`);
        
        return isValid;
    }
    
    // Validate and format IBAN
    static validateIBAN(iban) {
        // Remove spaces and convert to uppercase
        const cleaned = iban.replace(/\s/g, '').toUpperCase();
        
        // Check format: AE + 2 check digits + 16 digits
        const pattern = /^AE\d{18}$/;
        const isValid = pattern.test(cleaned);
        
        // Format with spaces (every 4 characters)
        const formatted = cleaned.match(/.{1,4}/g)?.join(' ') || cleaned;
        
        console.log(`IBAN Validation:`);
        console.log(`  Input: ${iban}`);
        console.log(`  Cleaned: ${cleaned}`);
        console.log(`  Formatted: ${formatted}`);
        console.log(`  Valid: ${isValid ? '✅' : '❌'}\n`);
        
        return { isValid, formatted };
    }
    
    // Mask credit card number
    static maskCreditCard(cardNumber) {
        // Remove all non-digits
        const digits = cardNumber.replace(/\D/g, '');
        
        // Validate length (13-19 digits)
        if (digits.length < 13 || digits.length > 19) {
            console.log(`Credit Card Masking:`);
            console.log(`  Input: ${cardNumber}`);
            console.log(`  Error: Invalid length\n`);
            return null;
        }
        
        // Mask all but last 4 digits
        const masked = digits.replace(/\d(?=\d{4})/g, '*');
        
        // Format with spaces (every 4 digits)
        const formatted = masked.match(/.{1,4}/g).join(' ');
        
        console.log(`Credit Card Masking:`);
        console.log(`  Input: ${cardNumber}`);
        console.log(`  Masked: ${formatted}\n`);
        
        return formatted;
    }
    
    // Validate and format phone number
    static validatePhone(phone) {
        // UAE mobile pattern
        const pattern = /^(\+971|0)?5[0-9]{1}[\s-]?\d{3}[\s-]?\d{4}$/;
        const isValid = pattern.test(phone);
        
        // Extract digits
        const digits = phone.replace(/\D/g, '');
        
        // Format: +971-5X-XXX-XXXX
        let formatted = phone;
        if (isValid && digits.length === 9) {
            formatted = `+971-${digits.substring(0, 2)}-${digits.substring(2, 5)}-${digits.substring(5)}`;
        } else if (isValid && digits.length === 12) {
            formatted = `+971-${digits.substring(3, 5)}-${digits.substring(5, 8)}-${digits.substring(8)}`;
        }
        
        console.log(`Phone Validation:`);
        console.log(`  Input: ${phone}`);
        console.log(`  Formatted: ${formatted}`);
        console.log(`  Valid: ${isValid ? '✅' : '❌'}\n`);
        
        return { isValid, formatted };
    }
    
    // Extract transaction details
    static extractTransactionDetails(text) {
        console.log(`Transaction Details Extraction:`);
        console.log(`  Input: ${text}\n`);
        
        // Extract account numbers
        const accountPattern = /ACC-\d{9}/g;
        const accounts = text.match(accountPattern);
        console.log(`  Accounts: ${accounts || 'none'}`);
        
        // Extract amounts
        const amountPattern = /AED\s?([\d,]+(?:\.\d{2})?)/g;
        const amounts = [...text.matchAll(amountPattern)].map(m => m[1]);
        console.log(`  Amounts: ${amounts.length > 0 ? amounts.join(', ') : 'none'}`);
        
        // Extract dates
        const datePattern = /\d{2}\/\d{2}\/\d{4}/g;
        const dates = text.match(datePattern);
        console.log(`  Dates: ${dates || 'none'}`);
        
        // Extract reference numbers
        const refPattern = /TXN-\d+/g;
        const references = text.match(refPattern);
        console.log(`  References: ${references || 'none'}\n`);
        
        return { accounts, amounts, dates, references };
    }
}

// Test validators
BankingValidator.validateAccountNumber('ACC-123456789');
BankingValidator.validateAccountNumber('ACC-12345');

BankingValidator.validateIBAN('AE070331234567890123456');
BankingValidator.validateIBAN('AE07 0331 2345 6789 0123 456');

BankingValidator.maskCreditCard('4111 1111 1111 1111');
BankingValidator.maskCreditCard('5500-0000-0000-0004');

BankingValidator.validatePhone('+971501234567');
BankingValidator.validatePhone('0501234567');

const transactionText = `
Transfer from ACC-111111111 to ACC-222222222
Amount: AED 5,000.00
Date: 15/03/2024
Reference: TXN-123456
`;
BankingValidator.extractTransactionDetails(transactionText);

console.log('='.repeat(70) + '\n');

// =============================================================================
// REGEX CHEAT SHEET
// =============================================================================

console.log('REGEX CHEAT SHEET:\n');

console.log('Character Classes:');
console.log('  \\d  - Digit [0-9]');
console.log('  \\D  - Non-digit');
console.log('  \\w  - Word character [a-zA-Z0-9_]');
console.log('  \\W  - Non-word character');
console.log('  \\s  - Whitespace');
console.log('  \\S  - Non-whitespace');
console.log('  .   - Any character (except newline)');

console.log('\nQuantifiers:');
console.log('  *   - 0 or more');
console.log('  +   - 1 or more');
console.log('  ?   - 0 or 1');
console.log('  {n} - Exactly n');
console.log('  {n,}- n or more');
console.log('  {n,m}- Between n and m');

console.log('\nAnchors:');
console.log('  ^   - Start of string');
console.log('  $   - End of string');
console.log('  \\b  - Word boundary');
console.log('  \\B  - Not word boundary');

console.log('\nGroups:');
console.log('  (x)    - Capturing group');
console.log('  (?:x)  - Non-capturing group');
console.log('  (?<name>x) - Named capturing group');
console.log('  (?=x)  - Positive lookahead');
console.log('  (?!x)  - Negative lookahead');

console.log('\nFlags:');
console.log('  g - Global (all matches)');
console.log('  i - Case insensitive');
console.log('  m - Multiline');
console.log('  s - Dotall (. matches \\n)');
console.log('  u - Unicode');
console.log('  y - Sticky');

console.log('\nCommon Patterns:');
console.log('  Email: /^[\\w.+-]+@[\\w.-]+\\.[a-zA-Z]{2,}$/');
console.log('  URL: /^https?:\\/\\/[\\w.-]+\\.\\w+/');
console.log('  Phone: /^\\+?\\d{1,3}[\\s-]?\\d{3,14}$/');
console.log('  Date: /^\\d{2}\\/\\d{2}\\/\\d{4}$/');
console.log('  IP: /^(\\d{1,3}\\.){3}\\d{1,3}$/');

console.log();
```

---

## Question 32: What are Symbols in JavaScript? Explain use cases and Symbol properties.

### Answer:

**Symbols** are a primitive data type introduced in ES6. They are unique and immutable identifiers, primarily used for creating unique property keys and implementing language-level metaprogramming.

### Key Features:
- **Unique**: Every Symbol is unique
- **Immutable**: Cannot be changed
- **Hidden**: Not enumerable by default
- **Well-known Symbols**: Built-in symbols for metaprogramming

### Banking Scenario: Using Symbols at Emirates NBD

```javascript
console.log('=== Symbols Demo - Emirates NBD ===\n');

// =============================================================================
// 1. SYMBOL BASICS
// =============================================================================

console.log('1. SYMBOL BASICS:\n');

// Creating symbols
const accountId = Symbol('accountId');
const transactionId = Symbol('transactionId');

// Each Symbol is unique
const symbol1 = Symbol('description');
const symbol2 = Symbol('description');

console.log('Symbol uniqueness:');
console.log(`  symbol1 === symbol2: ${symbol1 === symbol2}`);
console.log(`  typeof accountId: ${typeof accountId}`);
console.log(`  accountId.toString(): ${accountId.toString()}`);
console.log(`  accountId.description: ${accountId.description}\n`);

// =============================================================================
// 2. SYMBOL USE CASES
// =============================================================================

console.log('2. SYMBOL USE CASES:\n');

// Use Case 1: Private properties (convention)
class BankAccount {
    // Symbol properties are not truly private but harder to access
    constructor(accountNumber, balance, pin) {
        this.accountNumber = accountNumber; // Public
        this[Symbol.for('balance')] = balance; // "Private"
        this[Symbol.for('pin')] = pin; // "Private"
        this[Symbol.for('transactionHistory')] = [];
    }
    
    getBalance(inputPin) {
        if (inputPin === this[Symbol.for('pin')]) {
            return this[Symbol.for('balance')];
        }
        throw new Error('Invalid PIN');
    }
    
    deposit(amount, pin) {
        if (pin !== this[Symbol.for('pin')]) {
            throw new Error('Invalid PIN');
        }
        
        this[Symbol.for('balance')] += amount;
        this[Symbol.for('transactionHistory')].push({
            type: 'DEPOSIT',
            amount,
            timestamp: new Date()
        });
        
        return this[Symbol.for('balance')];
    }
}

const account = new BankAccount('ACC-123456789', 50000, '1234');

console.log('Symbol for "private" properties:');
console.log('  Account number (public):', account.accountNumber);
console.log('  Balance (symbol):', account.getBalance('1234'));
console.log('  Direct access to balance:', account[Symbol.for('balance')]);
console.log();

// Symbols don't appear in normal iteration
console.log('Property enumeration:');
console.log('  Object.keys():', Object.keys(account));
console.log('  for...in:');
for (let key in account) {
    console.log(`    ${key}`);
}
console.log();

// But can be retrieved with Object.getOwnPropertySymbols()
console.log('Symbol properties:');
const symbols = Object.getOwnPropertySymbols(account);
symbols.forEach(sym => {
    console.log(`  ${sym.toString()}: ${JSON.stringify(account[sym]).substring(0, 50)}`);
});
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. WELL-KNOWN SYMBOLS
// =============================================================================

console.log('3. WELL-KNOWN SYMBOLS:\n');

// Symbol.iterator - Make object iterable
class TransactionQueue {
    constructor() {
        this.transactions = [];
    }
    
    addTransaction(transaction) {
        this.transactions.push(transaction);
    }
    
    // Implement custom iteration
    [Symbol.iterator]() {
        let index = 0;
        const transactions = this.transactions;
        
        return {
            next() {
                if (index < transactions.length) {
                    return { value: transactions[index++], done: false };
                }
                return { done: true };
            }
        };
    }
}

const queue = new TransactionQueue();
queue.addTransaction({ id: 'TXN-001', amount: 5000 });
queue.addTransaction({ id: 'TXN-002', amount: 3000 });
queue.addTransaction({ id: 'TXN-003', amount: 7000 });

console.log('Symbol.iterator - Custom iteration:');
for (const txn of queue) {
    console.log(`  ${txn.id}: AED ${txn.amount}`);
}
console.log();

// Symbol.toStringTag - Custom object type
class SavingsAccount {
    constructor(balance) {
        this.balance = balance;
    }
    
    get [Symbol.toStringTag]() {
        return 'SavingsAccount';
    }
}

class CurrentAccount {
    constructor(balance) {
        this.balance = balance;
    }
    
    get [Symbol.toStringTag]() {
        return 'CurrentAccount';
    }
}

const savings = new SavingsAccount(100000);
const current = new CurrentAccount(50000);

console.log('Symbol.toStringTag - Custom type string:');
console.log(`  savings.toString(): ${savings.toString()}`);
console.log(`  current.toString(): ${current.toString()}`);
console.log(`  Object.prototype.toString.call(savings): ${Object.prototype.toString.call(savings)}\n`);

// Symbol.hasInstance - Custom instanceof behavior
class PremiumAccount {
    static [Symbol.hasInstance](instance) {
        return instance.balance >= 100000;
    }
}

console.log('Symbol.hasInstance - Custom instanceof:');
console.log(`  savings instanceof PremiumAccount: ${savings instanceof PremiumAccount}`);
console.log(`  current instanceof PremiumAccount: ${current instanceof PremiumAccount}\n`);

// Symbol.toPrimitive - Custom type conversion
class Money {
    constructor(amount, currency = 'AED') {
        this.amount = amount;
        this.currency = currency;
    }
    
    [Symbol.toPrimitive](hint) {
        console.log(`  Converting to ${hint}...`);
        
        if (hint === 'number') {
            return this.amount;
        }
        if (hint === 'string') {
            return `${this.currency} ${this.amount.toLocaleString()}`;
        }
        return this.amount; // default
    }
}

const balance = new Money(50000);

console.log('Symbol.toPrimitive - Custom conversion:');
console.log('  String context:', `Balance: ${balance}`);
console.log('  Number context:', +balance);
console.log('  Default context:', balance + 10000);
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. SYMBOL.FOR() - GLOBAL SYMBOL REGISTRY
// =============================================================================

console.log('4. GLOBAL SYMBOL REGISTRY:\n');

// Symbol.for() creates/retrieves global symbols
const globalSymbol1 = Symbol.for('transaction.type');
const globalSymbol2 = Symbol.for('transaction.type');

console.log('Symbol.for() - Global registry:');
console.log(`  globalSymbol1 === globalSymbol2: ${globalSymbol1 === globalSymbol2}`);
console.log(`  Symbol.keyFor(globalSymbol1): ${Symbol.keyFor(globalSymbol1)}\n`);

// Use case: Shared metadata between modules
class TransactionType {
    static DEPOSIT = Symbol.for('transaction.deposit');
    static WITHDRAWAL = Symbol.for('transaction.withdrawal');
    static TRANSFER = Symbol.for('transaction.transfer');
    
    static getName(type) {
        return Symbol.keyFor(type);
    }
}

const txnType = TransactionType.DEPOSIT;
console.log('Transaction type:');
console.log(`  Type: ${TransactionType.getName(txnType)}`);
console.log(`  Can be shared across modules/files\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PRACTICAL BANKING EXAMPLE
// =============================================================================

console.log('5. PRACTICAL BANKING EXAMPLE:\n');

// Secure account with symbol-based internal state
class SecureBankAccount {
    // Private symbols
    static #BALANCE = Symbol('balance');
    static #PIN = Symbol('pin');
    static #LOCKED = Symbol('locked');
    static #FAILED_ATTEMPTS = Symbol('failedAttempts');
    
    constructor(accountNumber, initialBalance, pin) {
        this.accountNumber = accountNumber;
        this[SecureBankAccount.#BALANCE] = initialBalance;
        this[SecureBankAccount.#PIN] = pin;
        this[SecureBankAccount.#LOCKED] = false;
        this[SecureBankAccount.#FAILED_ATTEMPTS] = 0;
    }
    
    // Custom iteration - iterate over transactions
    *[Symbol.iterator]() {
        // Would iterate over transactions in real implementation
        yield { type: 'OPENING', balance: this[SecureBankAccount.#BALANCE] };
    }
    
    // Custom object description
    get [Symbol.toStringTag]() {
        return 'SecureBankAccount';
    }
    
    // Verify PIN with security
    #verifyPin(inputPin) {
        if (this[SecureBankAccount.#LOCKED]) {
            throw new Error('Account locked due to multiple failed attempts');
        }
        
        if (inputPin !== this[SecureBankAccount.#PIN]) {
            this[SecureBankAccount.#FAILED_ATTEMPTS]++;
            
            if (this[SecureBankAccount.#FAILED_ATTEMPTS] >= 3) {
                this[SecureBankAccount.#LOCKED] = true;
                throw new Error('Account locked');
            }
            
            throw new Error('Invalid PIN');
        }
        
        this[SecureBankAccount.#FAILED_ATTEMPTS] = 0;
        return true;
    }
    
    // Public methods
    getBalance(pin) {
        this.#verifyPin(pin);
        return this[SecureBankAccount.#BALANCE];
    }
    
    deposit(amount, pin) {
        this.#verifyPin(pin);
        this[SecureBankAccount.#BALANCE] += amount;
        return this[SecureBankAccount.#BALANCE];
    }
    
    withdraw(amount, pin) {
        this.#verifyPin(pin);
        
        if (amount > this[SecureBankAccount.#BALANCE]) {
            throw new Error('Insufficient funds');
        }
        
        this[SecureBankAccount.#BALANCE] -= amount;
        return this[SecureBankAccount.#BALANCE];
    }
    
    isLocked() {
        return this[SecureBankAccount.#LOCKED];
    }
}

const secureAccount = new SecureBankAccount('ACC-987654321', 75000, '5678');

console.log('Secure account operations:\n');

try {
    console.log('1. Get balance (correct PIN):');
    console.log(`   Balance: AED ${secureAccount.getBalance('5678')}\n`);
    
    console.log('2. Deposit:');
    const newBalance = secureAccount.deposit(5000, '5678');
    console.log(`   New balance: AED ${newBalance}\n`);
    
    console.log('3. Try wrong PIN:');
    secureAccount.getBalance('0000');
} catch (error) {
    console.log(`   Error: ${error.message}\n`);
}

console.log('4. Account type:');
console.log(`   ${Object.prototype.toString.call(secureAccount)}\n`);

console.log('5. Symbol properties hidden:');
console.log(`   Enumerable keys: ${Object.keys(secureAccount)}`);
console.log(`   Symbol keys: ${Object.getOwnPropertySymbols(secureAccount).length} symbols\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// SYMBOL SUMMARY
// =============================================================================

console.log('SYMBOL SUMMARY:\n');

console.log('Key Features:');
console.log('  ✓ Unique identifiers');
console.log('  ✓ Immutable primitives');
console.log('  ✓ Not enumerable in for...in');
console.log('  ✓ Used for metaprogramming');

console.log('\nCommon Well-Known Symbols:');
console.log('  • Symbol.iterator - Custom iteration');
console.log('  • Symbol.toStringTag - Custom type string');
console.log('  • Symbol.hasInstance - Custom instanceof');
console.log('  • Symbol.toPrimitive - Custom type conversion');
console.log('  • Symbol.for() - Global symbol registry');

console.log('\nUse Cases:');
console.log('  1. "Private" object properties');
console.log('  2. Prevent property name collisions');
console.log('  3. Implement custom iteration');
console.log('  4. Define metadata');
console.log('  5. Framework/library internal values');

console.log();
```

---

## Question 33: What is the Reflect API? How does it differ from Object methods?

### Answer:

The **Reflect API** provides methods for interceptable JavaScript operations. It's a built-in object that provides methods for performing meta-operations on objects, similar to Proxy traps.

### Key Differences from Object:
- **Consistent API**: All methods return predictable values
- **Function behavior**: Works well with Proxy handlers
- **Better error handling**: Returns boolean for success/failure
- **Cleaner syntax**: More intuitive than Object methods

### Banking Scenario: Reflect API at Emirates NBD

```javascript
console.log('=== Reflect API Demo - Emirates NBD ===\n');

// =============================================================================
// 1. REFLECT VS OBJECT METHODS
// =============================================================================

console.log('1. REFLECT VS OBJECT METHODS:\n');

const account = {
    accountNumber: 'ACC-123456789',
    balance: 50000,
    type: 'SAVINGS'
};

console.log('Comparison: Reflect vs Object\n');

// Property access
console.log('Property Access:');
console.log(`  Reflect.get(): ${Reflect.get(account, 'balance')}`);
console.log(`  Direct access: ${account.balance}\n`);

// Property setting
console.log('Property Setting:');
const setSuccess = Reflect.set(account, 'balance', 55000);
console.log(`  Reflect.set() success: ${setSuccess}`);
console.log(`  New balance: ${account.balance}\n`);

// Property check
console.log('Property Check:');
console.log(`  Reflect.has(): ${Reflect.has(account, 'balance')}`);
console.log(`  'in' operator: ${'balance' in account}\n`);

// Property deletion
console.log('Property Deletion:');
Reflect.defineProperty(account, 'temp', { value: 'test', configurable: true });
const deleteSuccess = Reflect.deleteProperty(account, 'temp');
console.log(`  Reflect.deleteProperty() success: ${deleteSuccess}\n`);

// Get property descriptor
console.log('Property Descriptor:');
const descriptor = Reflect.getOwnPropertyDescriptor(account, 'balance');
console.log(`  Reflect.getOwnPropertyDescriptor():`, descriptor);
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. REFLECT METHODS
// =============================================================================

console.log('2. REFLECT METHODS:\n');

class ReflectDemo {
    static demonstrateMethods() {
        const bankAccount = {
            accountNumber: 'ACC-123456789',
            _balance: 50000,
            type: 'SAVINGS',
            
            get balance() {
                return this._balance;
            },
            
            set balance(value) {
                if (value < 0) {
                    throw new Error('Balance cannot be negative');
                }
                this._balance = value;
            },
            
            deposit(amount) {
                this._balance += amount;
                return this._balance;
            }
        };
        
        // 1. Reflect.get(target, propertyKey, [receiver])
        console.log('1. Reflect.get():');
        console.log(`   Balance: ${Reflect.get(bankAccount, 'balance')}`);
        console.log(`   Account: ${Reflect.get(bankAccount, 'accountNumber')}\n`);
        
        // 2. Reflect.set(target, propertyKey, value, [receiver])
        console.log('2. Reflect.set():');
        const success = Reflect.set(bankAccount, 'type', 'CURRENT');
        console.log(`   Set type success: ${success}`);
        console.log(`   New type: ${bankAccount.type}\n`);
        
        // 3. Reflect.has(target, propertyKey)
        console.log('3. Reflect.has():');
        console.log(`   Has 'balance': ${Reflect.has(bankAccount, 'balance')}`);
        console.log(`   Has 'customer': ${Reflect.has(bankAccount, 'customer')}\n`);
        
        // 4. Reflect.deleteProperty(target, propertyKey)
        console.log('4. Reflect.deleteProperty():');
        bankAccount.tempProperty = 'temporary';
        const deleted = Reflect.deleteProperty(bankAccount, 'tempProperty');
        console.log(`   Delete success: ${deleted}`);
        console.log(`   Property exists: ${Reflect.has(bankAccount, 'tempProperty')}\n`);
        
        // 5. Reflect.apply(target, thisArg, argumentsList)
        console.log('5. Reflect.apply():');
        const newBalance = Reflect.apply(bankAccount.deposit, bankAccount, [5000]);
        console.log(`   Deposit result: ${newBalance}\n`);
        
        // 6. Reflect.construct(target, argumentsList, [newTarget])
        console.log('6. Reflect.construct():');
        class Transaction {
            constructor(id, amount) {
                this.id = id;
                this.amount = amount;
            }
        }
        const txn = Reflect.construct(Transaction, ['TXN-001', 5000]);
        console.log(`   Created transaction:`, txn);
        console.log();
        
        // 7. Reflect.defineProperty(target, propertyKey, attributes)
        console.log('7. Reflect.defineProperty():');
        const defined = Reflect.defineProperty(bankAccount, 'currency', {
            value: 'AED',
            writable: false,
            enumerable: true,
            configurable: false
        });
        console.log(`   Define success: ${defined}`);
        console.log(`   Currency: ${bankAccount.currency}\n`);
        
        // 8. Reflect.getOwnPropertyDescriptor(target, propertyKey)
        console.log('8. Reflect.getOwnPropertyDescriptor():');
        const desc = Reflect.getOwnPropertyDescriptor(bankAccount, 'currency');
        console.log(`   Descriptor:`, desc);
        console.log();
        
        // 9. Reflect.getPrototypeOf(target)
        console.log('9. Reflect.getPrototypeOf():');
        const proto = Reflect.getPrototypeOf(bankAccount);
        console.log(`   Prototype: ${proto.constructor.name}\n`);
        
        // 10. Reflect.setPrototypeOf(target, prototype)
        console.log('10. Reflect.setPrototypeOf():');
        const newProto = { customMethod() { return 'custom'; } };
        const protoSet = Reflect.setPrototypeOf({}, newProto);
        console.log(`   Set prototype success: ${protoSet}\n`);
        
        // 11. Reflect.isExtensible(target)
        console.log('11. Reflect.isExtensible():');
        console.log(`   Is extensible: ${Reflect.isExtensible(bankAccount)}`);
        Reflect.preventExtensions(bankAccount);
        console.log(`   After preventExtensions: ${Reflect.isExtensible(bankAccount)}\n`);
        
        // 12. Reflect.ownKeys(target)
        console.log('12. Reflect.ownKeys():');
        const keys = Reflect.ownKeys(bankAccount);
        console.log(`   Own keys:`, keys);
        console.log();
    }
}

ReflectDemo.demonstrateMethods();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. REFLECT WITH PROXY
// =============================================================================

console.log('3. REFLECT WITH PROXY:\n');

// Secure account with Reflect and Proxy
class SecureAccountProxy {
    static create(accountNumber, balance, pin) {
        const target = {
            accountNumber,
            balance,
            pin,
            transactions: [],
            locked: false
        };
        
        const handler = {
            get(target, prop, receiver) {
                // Log access
                console.log(`  [GET] Accessing property: ${String(prop)}`);
                
                // Prevent direct pin access
                if (prop === 'pin') {
                    console.log(`  [DENIED] Cannot access PIN directly`);
                    return undefined;
                }
                
                // Use Reflect for normal behavior
                return Reflect.get(target, prop, receiver);
            },
            
            set(target, prop, value, receiver) {
                console.log(`  [SET] Setting ${String(prop)} = ${value}`);
                
                // Validate balance
                if (prop === 'balance') {
                    if (typeof value !== 'number' || value < 0) {
                        console.log(`  [DENIED] Invalid balance value`);
                        return false;
                    }
                }
                
                // Prevent pin modification
                if (prop === 'pin') {
                    console.log(`  [DENIED] Cannot modify PIN directly`);
                    return false;
                }
                
                // Use Reflect for normal behavior
                return Reflect.set(target, prop, value, receiver);
            },
            
            has(target, prop) {
                console.log(`  [HAS] Checking property: ${String(prop)}`);
                
                // Hide sensitive properties
                if (prop === 'pin') {
                    return false;
                }
                
                return Reflect.has(target, prop);
            },
            
            deleteProperty(target, prop) {
                console.log(`  [DELETE] Attempting to delete: ${String(prop)}`);
                
                // Prevent deletion of critical properties
                if (['accountNumber', 'balance', 'pin'].includes(prop)) {
                    console.log(`  [DENIED] Cannot delete critical property`);
                    return false;
                }
                
                return Reflect.deleteProperty(target, prop);
            }
        };
        
        return new Proxy(target, handler);
    }
}

console.log('Creating secure account proxy:\n');
const proxyAccount = SecureAccountProxy.create('ACC-987654321', 100000, '9876');

console.log('\nOperations:\n');

console.log('1. Read balance:');
console.log(`   Value: ${proxyAccount.balance}\n`);

console.log('2. Try to read PIN:');
console.log(`   Value: ${proxyAccount.pin}\n`);

console.log('3. Set balance:');
proxyAccount.balance = 105000;
console.log(`   New balance: ${proxyAccount.balance}\n`);

console.log('4. Try to set invalid balance:');
proxyAccount.balance = -1000;
console.log();

console.log('5. Check property existence:');
console.log(`   'balance' in account: ${'balance' in proxyAccount}`);
console.log(`   'pin' in account: ${'pin' in proxyAccount}\n`);

console.log('6. Try to delete balance:');
delete proxyAccount.balance;
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. PRACTICAL BANKING EXAMPLE
// =============================================================================

console.log('4. PRACTICAL BANKING EXAMPLE:\n');

// Audit logging with Reflect
class AuditedBankAccount {
    constructor(accountNumber, balance) {
        this.accountNumber = accountNumber;
        this._balance = balance;
        this.auditLog = [];
        
        return new Proxy(this, {
            get(target, prop, receiver) {
                // Log all property access
                if (prop !== 'auditLog' && prop !== '_balance') {
                    Reflect.get(target, 'auditLog').push({
                        operation: 'GET',
                        property: prop,
                        timestamp: new Date()
                    });
                }
                
                return Reflect.get(target, prop, receiver);
            },
            
            set(target, prop, value, receiver) {
                // Log all modifications
                if (prop !== 'auditLog') {
                    const oldValue = Reflect.get(target, prop);
                    Reflect.get(target, 'auditLog').push({
                        operation: 'SET',
                        property: prop,
                        oldValue,
                        newValue: value,
                        timestamp: new Date()
                    });
                }
                
                return Reflect.set(target, prop, value, receiver);
            }
        });
    }
    
    get balance() {
        return this._balance;
    }
    
    set balance(value) {
        if (value < 0) throw new Error('Balance cannot be negative');
        this._balance = value;
    }
    
    deposit(amount) {
        this.balance += amount;
        return this.balance;
    }
    
    withdraw(amount) {
        if (amount > this.balance) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
        return this.balance;
    }
    
    getAuditLog() {
        return this.auditLog;
    }
}

const auditedAccount = new AuditedBankAccount('ACC-555555555', 50000);

console.log('Audited account operations:\n');

console.log('1. Check balance:');
console.log(`   Balance: AED ${auditedAccount.balance}\n`);

console.log('2. Deposit:');
auditedAccount.deposit(10000);
console.log(`   New balance: AED ${auditedAccount.balance}\n`);

console.log('3. Withdraw:');
auditedAccount.withdraw(5000);
console.log(`   New balance: AED ${auditedAccount.balance}\n`);

console.log('4. Audit log:');
const log = auditedAccount.getAuditLog();
log.slice(0, 5).forEach((entry, i) => {
    console.log(`   ${i + 1}. ${entry.operation} ${entry.property}`);
    if (entry.oldValue !== undefined) {
        console.log(`      ${entry.oldValue} → ${entry.newValue}`);
    }
});
console.log(`   ... ${log.length} total entries\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// REFLECT API SUMMARY
// =============================================================================

console.log('REFLECT API SUMMARY:\n');

console.log('All Reflect Methods:');
console.log('  1.  Reflect.get(target, prop, [receiver])');
console.log('  2.  Reflect.set(target, prop, value, [receiver])');
console.log('  3.  Reflect.has(target, prop)');
console.log('  4.  Reflect.deleteProperty(target, prop)');
console.log('  5.  Reflect.apply(fn, thisArg, args)');
console.log('  6.  Reflect.construct(constructor, args, [newTarget])');
console.log('  7.  Reflect.defineProperty(target, prop, descriptor)');
console.log('  8.  Reflect.getOwnPropertyDescriptor(target, prop)');
console.log('  9.  Reflect.getPrototypeOf(target)');
console.log('  10. Reflect.setPrototypeOf(target, prototype)');
console.log('  11. Reflect.isExtensible(target)');
console.log('  12. Reflect.preventExtensions(target)');
console.log('  13. Reflect.ownKeys(target)');

console.log('\nAdvantages over Object methods:');
console.log('  ✓ Consistent return values (boolean for success)');
console.log('  ✓ Better error handling');
console.log('  ✓ Functional programming friendly');
console.log('  ✓ Perfect match for Proxy handlers');
console.log('  ✓ More intuitive API');

console.log('\nCommon Use Cases:');
console.log('  1. Implementing Proxy handlers');
console.log('  2. Metaprogramming and reflection');
console.log('  3. Dynamic property access');
console.log('  4. Validation and security');
console.log('  5. Audit logging');
console.log('  6. Framework development');

console.log();
```

### Key Takeaways:

**Regular Expressions**:
- Pattern matching and validation
- Flags: g, i, m, s, u, y
- Methods: test(), exec(), match(), replace(), split()
- Essential for data validation in banking

**Symbols**:
- Unique, immutable identifiers
- "Private" properties (convention)
- Well-known symbols for metaprogramming
- Symbol.iterator, Symbol.toStringTag, etc.

**Reflect API**:
- Consistent meta-operations
- Works with Proxy handlers
- Better error handling than Object methods
- Essential for metaprogramming

---

**End of Questions 31-33**
