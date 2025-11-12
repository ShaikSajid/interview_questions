# Questions 37-39: Data Types, Type Coercion & Scope Deep Dive

## Question 37: Explain JavaScript data types - primitive vs reference types, and type coercion

### Answer:

JavaScript has **7 primitive types** and **1 reference type (Object)**. Understanding the difference and how type coercion works is fundamental for avoiding bugs.

### Primitive Types:
1. **string** - Text data
2. **number** - Numeric values (integers and floats)
3. **boolean** - true/false
4. **null** - Intentional absence of value
5. **undefined** - Variable declared but not assigned
6. **symbol** - Unique identifier (ES6)
7. **bigint** - Large integers (ES2020)

### Reference Type:
- **object** - Collections of properties (includes arrays, functions, dates, etc.)

### Banking Scenario: Data Types at Emirates NBD

```javascript
console.log('=== JavaScript Data Types - Emirates NBD ===\n');

// =============================================================================
// 1. PRIMITIVE TYPES
// =============================================================================

console.log('1. PRIMITIVE TYPES:\n');

// String
const accountHolder = 'Ahmed Al Maktoum';
const accountNumber = 'ACC-123456789';

console.log('String:');
console.log(`  Account holder: ${accountHolder} (type: ${typeof accountHolder})`);
console.log(`  Account number: ${accountNumber}\n`);

// Number (JavaScript has only one number type - IEEE 754 floating point)
const balance = 50000.50;
const transactionLimit = 100000;
const interestRate = 0.035; // 3.5%
const negativeBalance = -500;
const infiniteValue = Infinity;
const notANumber = NaN;

console.log('Number:');
console.log(`  Balance: ${balance} (type: ${typeof balance})`);
console.log(`  Transaction limit: ${transactionLimit}`);
console.log(`  Interest rate: ${interestRate}`);
console.log(`  Negative: ${negativeBalance}`);
console.log(`  Infinity: ${infiniteValue} (type: ${typeof infiniteValue})`);
console.log(`  NaN: ${notANumber} (type: ${typeof notANumber})`);
console.log(`  NaN === NaN: ${NaN === NaN} (always false!)`);
console.log(`  Number.isNaN(NaN): ${Number.isNaN(NaN)}\n`);

// Boolean
const isActive = true;
const isFrozen = false;
const hasOverdraft = true;

console.log('Boolean:');
console.log(`  Account active: ${isActive} (type: ${typeof isActive})`);
console.log(`  Account frozen: ${isFrozen}\n`);

// Null - intentional absence of value
const closedAccountBalance = null;

console.log('Null:');
console.log(`  Closed account: ${closedAccountBalance} (type: ${typeof closedAccountBalance})`);
console.log(`  ⚠️  typeof null returns "object" - JavaScript bug!\n`);

// Undefined - variable declared but not initialized
let pendingTransaction;
let uninitializedBalance;

console.log('Undefined:');
console.log(`  Pending transaction: ${pendingTransaction} (type: ${typeof pendingTransaction})`);
console.log(`  Uninitialized: ${uninitializedBalance}\n`);

// Symbol - unique identifier
const accountId = Symbol('accountId');
const internalId = Symbol('accountId');

console.log('Symbol:');
console.log(`  Account ID: ${accountId.toString()} (type: ${typeof accountId})`);
console.log(`  Symbol('accountId') === Symbol('accountId'): ${accountId === internalId}`);
console.log(`  Each Symbol is unique!\n`);

// BigInt - large integers
const largeAccountNumber = 123456789012345678901234567890n;
const transactionCount = BigInt('999999999999999999');

console.log('BigInt:');
console.log(`  Large account: ${largeAccountNumber} (type: ${typeof largeAccountNumber})`);
console.log(`  Transaction count: ${transactionCount}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. REFERENCE TYPES (OBJECTS)
// =============================================================================

console.log('2. REFERENCE TYPES:\n');

// Object literal
const account = {
    accountNumber: 'ACC-123456789',
    balance: 50000,
    holder: 'Ahmed Al Maktoum'
};

// Array (special type of object)
const transactions = [
    { id: 1, amount: 5000, type: 'DEPOSIT' },
    { id: 2, amount: 2000, type: 'WITHDRAWAL' }
];

// Function (special type of object)
function calculateInterest(balance, rate) {
    return balance * rate;
}

// Date (built-in object)
const accountOpenDate = new Date('2024-01-01');

// RegExp (built-in object)
const accountPattern = /^ACC-\d{9}$/;

console.log('Object types:');
console.log(`  Object: ${typeof account} - ${JSON.stringify(account)}`);
console.log(`  Array: ${typeof transactions} (${Array.isArray(transactions) ? 'is array' : 'not array'})`);
console.log(`  Function: ${typeof calculateInterest}`);
console.log(`  Date: ${typeof accountOpenDate} (${accountOpenDate instanceof Date ? 'is Date' : 'not Date'})`);
console.log(`  RegExp: ${typeof accountPattern} (${accountPattern instanceof RegExp ? 'is RegExp' : 'not RegExp'})\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. PRIMITIVE VS REFERENCE - KEY DIFFERENCES
// =============================================================================

console.log('3. PRIMITIVE VS REFERENCE:\n');

console.log('Primitives - Stored by VALUE:\n');

// Primitives are copied by value
let balance1 = 10000;
let balance2 = balance1; // Copy the value

balance1 = 15000; // Changing balance1 doesn't affect balance2

console.log(`  balance1 = ${balance1}`);
console.log(`  balance2 = ${balance2}`);
console.log('  ✓ Independent values\n');

console.log('Objects - Stored by REFERENCE:\n');

// Objects are copied by reference
const account1 = { balance: 10000 };
const account2 = account1; // Copy the reference (both point to same object)

account1.balance = 15000; // Changing via account1 affects account2

console.log(`  account1.balance = ${account1.balance}`);
console.log(`  account2.balance = ${account2.balance}`);
console.log('  ⚠️  Both point to same object!\n');

// Creating a true copy
const account3 = { ...account1 }; // Shallow copy
account3.balance = 20000;

console.log('  Shallow copy with spread operator:');
console.log(`  account1.balance = ${account1.balance}`);
console.log(`  account3.balance = ${account3.balance}`);
console.log('  ✓ Now independent\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. TYPE COERCION
// =============================================================================

console.log('4. TYPE COERCION:\n');

console.log('Implicit Coercion Examples:\n');

// String coercion (+ operator)
const accountNumber1 = 'ACC-';
const accountNumber2 = 123456789;
console.log(`  String + Number: "${accountNumber1}" + ${accountNumber2} = "${accountNumber1 + accountNumber2}"`);
console.log('  → Number coerced to string\n');

// Numeric coercion (arithmetic operators)
const balance3 = '5000';
const withdrawal = '1000';
console.log(`  String - String: "${balance3}" - "${withdrawal}" = ${balance3 - withdrawal}`);
console.log('  → Strings coerced to numbers\n');

const amount = '100';
console.log(`  String * 2: "${amount}" * 2 = ${amount * 2}`);
console.log(`  String / 2: "${amount}" / 2 = ${amount / 2}`);
console.log('  → String coerced to number\n');

// Boolean coercion
console.log('Boolean Coercion:\n');

const emptyString = '';
const zeroBalance = 0;
const nullValue = null;
const undefinedValue = undefined;
const nanValue = NaN;
const fullBalance = 5000;

console.log('  Falsy values (coerce to false):');
console.log(`    Boolean(""): ${Boolean(emptyString)}`);
console.log(`    Boolean(0): ${Boolean(zeroBalance)}`);
console.log(`    Boolean(null): ${Boolean(nullValue)}`);
console.log(`    Boolean(undefined): ${Boolean(undefinedValue)}`);
console.log(`    Boolean(NaN): ${Boolean(nanValue)}`);
console.log(`    Boolean(false): ${Boolean(false)}\n`);

console.log('  Truthy values (coerce to true):');
console.log(`    Boolean(5000): ${Boolean(fullBalance)}`);
console.log(`    Boolean("text"): ${Boolean('text')}`);
console.log(`    Boolean([]): ${Boolean([])}`);
console.log(`    Boolean({}): ${Boolean({})}`);
console.log(`    Boolean(true): ${Boolean(true)}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. == VS === (EQUALITY OPERATORS)
// =============================================================================

console.log('5. == VS === (LOOSE VS STRICT EQUALITY):\n');

console.log('== (Loose Equality) - Performs Type Coercion:\n');

console.log(`  5 == "5": ${5 == "5"} (number vs string)`);
console.log(`  0 == false: ${0 == false} (number vs boolean)`);
console.log(`  "" == false: ${'' == false} (string vs boolean)`);
console.log(`  null == undefined: ${null == undefined} (special case)`);
console.log(`  [] == false: ${[] == false} (object vs boolean)`);
console.log(`  "0" == false: ${'0' == false} (string vs boolean)\n`);

console.log('=== (Strict Equality) - NO Type Coercion:\n');

console.log(`  5 === "5": ${5 === "5"} (different types)`);
console.log(`  0 === false: ${0 === false} (different types)`);
console.log(`  "" === false: ${'' === false} (different types)`);
console.log(`  null === undefined: ${null === undefined} (different types)`);
console.log(`  5 === 5: ${5 === 5} (same type and value)`);
console.log(`  "hello" === "hello": ${'hello' === 'hello'} (same type and value)\n`);

console.log('⚠️  BANKING EXAMPLES - WHY === IS IMPORTANT:\n');

class BankingValidator {
    // ❌ BAD: Using ==
    static validateAmountBad(amount) {
        if (amount == 0) { // "0" == 0 returns true!
            return false;
        }
        return true;
    }
    
    // ✅ GOOD: Using ===
    static validateAmountGood(amount) {
        if (amount === 0) { // "0" === 0 returns false
            return false;
        }
        return true;
    }
}

console.log('Validating amount with "0" (string):');
console.log(`  Using ==: ${BankingValidator.validateAmountBad('0')} (incorrect!)`);
console.log(`  Using ===: ${BankingValidator.validateAmountGood('0')} (correct!)\n`);

console.log('Validating amount with 0 (number):');
console.log(`  Using ==: ${BankingValidator.validateAmountBad(0)}`);
console.log(`  Using ===: ${BankingValidator.validateAmountGood(0)}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. PRACTICAL BANKING TYPE CHECKING
// =============================================================================

console.log('6. PRACTICAL TYPE CHECKING:\n');

class TypeChecker {
    // Comprehensive type checking
    static getType(value) {
        // Handle null specially (typeof null === "object" is a bug)
        if (value === null) return 'null';
        
        // Handle arrays specially
        if (Array.isArray(value)) return 'array';
        
        // Handle dates
        if (value instanceof Date) return 'date';
        
        // Handle regular expressions
        if (value instanceof RegExp) return 'regexp';
        
        // For everything else, use typeof
        return typeof value;
    }
    
    // Check if value is a valid number
    static isValidNumber(value) {
        return typeof value === 'number' && !isNaN(value) && isFinite(value);
    }
    
    // Check if value is a valid string
    static isValidString(value) {
        return typeof value === 'string' && value.trim().length > 0;
    }
    
    // Check if value is a plain object
    static isPlainObject(value) {
        return value !== null && 
               typeof value === 'object' && 
               !Array.isArray(value) &&
               !(value instanceof Date) &&
               !(value instanceof RegExp);
    }
}

// Test type checker
const testValues = [
    { label: 'Number', value: 5000 },
    { label: 'String', value: 'ACC-123' },
    { label: 'Boolean', value: true },
    { label: 'Null', value: null },
    { label: 'Undefined', value: undefined },
    { label: 'Array', value: [1, 2, 3] },
    { label: 'Object', value: { balance: 5000 } },
    { label: 'Date', value: new Date() },
    { label: 'RegExp', value: /test/ },
    { label: 'NaN', value: NaN },
    { label: 'Infinity', value: Infinity }
];

console.log('Type Detection Results:\n');
testValues.forEach(({ label, value }) => {
    const type = TypeChecker.getType(value);
    const isValid = TypeChecker.isValidNumber(value);
    console.log(`  ${label.padEnd(12)} → ${type.padEnd(10)} (valid number: ${isValid})`);
});
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 7. COMMON TYPE COERCION PITFALLS IN BANKING
// =============================================================================

console.log('7. COMMON PITFALLS IN BANKING:\n');

console.log('Pitfall 1: String concatenation instead of addition:\n');

const input1 = '5000'; // From user input
const input2 = '3000'; // From user input

console.log(`  ❌ "${input1}" + "${input2}" = "${input1 + input2}" (concatenation!)`);
console.log(`  ✅ Number("${input1}") + Number("${input2}") = ${Number(input1) + Number(input2)} (addition)\n`);

console.log('Pitfall 2: Comparing different types:\n');

const balance4 = 0;
const balanceStr = '0';

console.log(`  ❌ ${balance4} == "${balanceStr}": ${balance4 == balanceStr} (coercion)`);
console.log(`  ✅ ${balance4} === Number("${balanceStr}"): ${balance4 === Number(balanceStr)} (explicit)\n`);

console.log('Pitfall 3: Falsy values in conditions:\n');

function processTransaction(amount) {
    // ❌ BAD: 0 is falsy
    if (!amount) {
        return 'Invalid amount';
    }
    return `Processing ${amount}`;
}

function processTransactionFixed(amount) {
    // ✅ GOOD: Explicit check
    if (typeof amount !== 'number' || isNaN(amount)) {
        return 'Invalid amount';
    }
    if (amount <= 0) {
        return 'Amount must be positive';
    }
    return `Processing ${amount}`;
}

console.log(`  ❌ processTransaction(0): "${processTransaction(0)}" (rejects valid zero)`);
console.log(`  ✅ processTransactionFixed(0): "${processTransactionFixed(0)}" (correct check)\n`);

console.log('Pitfall 4: Array/Object in boolean context:\n');

const emptyArray = [];
const emptyObject = {};

console.log(`  Boolean([]): ${Boolean(emptyArray)} (empty array is truthy!)`);
console.log(`  Boolean({}): ${Boolean(emptyObject)} (empty object is truthy!)`);
console.log(`  Correct check: [].length === 0: ${emptyArray.length === 0}`);
console.log(`  Correct check: Object.keys({}).length === 0: ${Object.keys(emptyObject).length === 0}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 8. TYPE CONVERSION METHODS
// =============================================================================

console.log('8. TYPE CONVERSION METHODS:\n');

console.log('String Conversion:\n');
console.log(`  String(5000): "${String(5000)}"`);
console.log(`  (5000).toString(): "${(5000).toString()}"`);
console.log(`  5000 + "": "${5000 + ''}"`);

console.log('\nNumber Conversion:\n');
console.log(`  Number("5000"): ${Number('5000')}`);
console.log(`  Number("5000.50"): ${Number('5000.50')}`);
console.log(`  parseInt("5000"): ${parseInt('5000')}`);
console.log(`  parseInt("5000.99"): ${parseInt('5000.99')} (truncates decimals)`);
console.log(`  parseFloat("5000.99"): ${parseFloat('5000.99')}`);
console.log(`  +"5000": ${+'5000'} (unary plus operator)`);

console.log('\nBoolean Conversion:\n');
console.log(`  Boolean(1): ${Boolean(1)}`);
console.log(`  Boolean(0): ${Boolean(0)}`);
console.log(`  !!1: ${!!1} (double negation)`);
console.log(`  !!0: ${!!0}`);

console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// DATA TYPES SUMMARY
// =============================================================================

console.log('DATA TYPES SUMMARY:\n');

console.log('Primitive Types (7):');
console.log('  1. string    - Immutable text');
console.log('  2. number    - 64-bit floating point');
console.log('  3. boolean   - true/false');
console.log('  4. null      - Intentional absence');
console.log('  5. undefined - Uninitialized');
console.log('  6. symbol    - Unique identifier');
console.log('  7. bigint    - Large integers');

console.log('\nReference Type:');
console.log('  object      - Collections (arrays, functions, dates, etc.)');

console.log('\nKey Differences:');
console.log('  Primitives:');
console.log('    ✓ Stored by value');
console.log('    ✓ Immutable');
console.log('    ✓ Compared by value');
console.log('    ✓ Copied by value');

console.log('\n  Objects:');
console.log('    ✓ Stored by reference');
console.log('    ✓ Mutable');
console.log('    ✓ Compared by reference');
console.log('    ✓ Copied by reference');

console.log('\nType Coercion Rules:');
console.log('  • + operator: Converts to string if either operand is string');
console.log('  • - * / operators: Convert to number');
console.log('  • == operator: Performs type coercion');
console.log('  • === operator: NO type coercion (strict equality)');

console.log('\nBest Practices:');
console.log('  ✓ Always use === instead of ==');
console.log('  ✓ Explicitly convert types when needed');
console.log('  ✓ Validate input types in functions');
console.log('  ✓ Use typeof for primitives, instanceof for objects');
console.log('  ✓ Check for NaN with Number.isNaN()');
console.log('  ✓ Be careful with falsy values (0, "", null, undefined, NaN, false)');

console.log();
```

---

## Question 38: Explain scope in JavaScript - global, function, block, and lexical scope

### Answer:

**Scope** determines the accessibility of variables, functions, and objects in different parts of your code. JavaScript has multiple scope levels that affect variable visibility and lifetime.

### Types of Scope:
1. **Global Scope** - Accessible everywhere
2. **Function Scope** - Accessible within function (var)
3. **Block Scope** - Accessible within block {} (let/const)
4. **Lexical Scope** - Inner functions access outer scope

### Banking Scenario: Scope Management at Emirates NBD

```javascript
console.log('=== JavaScript Scope - Emirates NBD ===\n');

// =============================================================================
// 1. GLOBAL SCOPE
// =============================================================================

console.log('1. GLOBAL SCOPE:\n');

// Global variables - accessible everywhere
var bankName = 'Emirates NBD'; // Global (not recommended)
let bankCode = 'ENBD'; // Global
const bankCountry = 'UAE'; // Global

function displayBankInfo() {
    console.log(`  Inside function: ${bankName} (${bankCode}) - ${bankCountry}`);
}

displayBankInfo();
console.log(`  Outside function: ${bankName} (${bankCode}) - ${bankCountry}\n`);

console.log('⚠️  Global variables can be accessed and modified anywhere\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. FUNCTION SCOPE (VAR)
// =============================================================================

console.log('2. FUNCTION SCOPE (VAR):\n');

function processTransaction() {
    var transactionId = 'TXN-001'; // Function-scoped
    var amount = 5000;
    
    console.log(`  Inside function: ${transactionId}, Amount: ${amount}`);
    
    if (amount > 1000) {
        var fee = 10; // Still function-scoped (not block-scoped!)
        console.log(`  Inside if block: Fee = ${fee}`);
    }
    
    console.log(`  After if block: Fee = ${fee} (accessible!)`);
    console.log('  ⚠️  var is function-scoped, not block-scoped\n');
}

processTransaction();

// console.log(transactionId); // ReferenceError: transactionId is not defined

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. BLOCK SCOPE (LET/CONST)
// =============================================================================

console.log('3. BLOCK SCOPE (LET/CONST):\n');

function processTransactionWithBlockScope() {
    let transactionId = 'TXN-002'; // Block-scoped
    const maxAmount = 100000; // Block-scoped
    
    console.log(`  Function level: ${transactionId}, Max: ${maxAmount}`);
    
    if (true) {
        let blockVariable = 'only in block'; // Block-scoped
        const blockConstant = 'cannot escape'; // Block-scoped
        var functionVariable = 'accessible outside'; // Function-scoped
        
        console.log(`  Inside block: ${blockVariable}`);
    }
    
    // console.log(blockVariable); // ReferenceError
    console.log(`  Outside block: ${functionVariable} (var escapes block)`);
    console.log('  ✓ let/const are block-scoped\n');
}

processTransactionWithBlockScope();

// Practical example - loop scope
console.log('Loop Variable Scope:\n');

console.log('Using var (function-scoped):');
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(`  Loop var i = ${i}`), 10);
}
setTimeout(() => {
    console.log(`  After loop, var i = ${i} (accessible outside!)\n`);
}, 20);

setTimeout(() => {
    console.log('Using let (block-scoped):');
    for (let j = 0; j < 3; j++) {
        setTimeout(() => console.log(`  Loop let j = ${j}`), 10);
    }
    setTimeout(() => {
        // console.log(j); // ReferenceError: j is not defined
        console.log('  After loop, let j is not accessible outside\n');
    }, 20);
}, 50);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 100);

// =============================================================================
// 4. LEXICAL SCOPE (CLOSURES)
// =============================================================================

setTimeout(() => {
    console.log('4. LEXICAL SCOPE:\n');
    
    // Outer function
    function createAccount(accountHolder, initialBalance) {
        // Outer scope variables
        const accountNumber = `ACC-${Date.now()}`;
        let balance = initialBalance;
        
        console.log(`  Outer scope: Created account ${accountNumber}`);
        
        // Inner function has access to outer scope (lexical scope)
        function deposit(amount) {
            balance += amount; // Accesses outer scope variable
            console.log(`  Inner scope: Deposited ${amount}, Balance: ${balance}`);
            return balance;
        }
        
        function getBalance() {
            return balance; // Accesses outer scope variable
        }
        
        // Return inner functions (closures)
        return { deposit, getBalance, accountNumber };
    }
    
    const account = createAccount('Ahmed', 10000);
    account.deposit(5000);
    console.log(`  Final balance: ${account.getBalance()}\n`);
    
    console.log('  ✓ Inner functions access outer scope variables\n');
}, 120);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 140);

// =============================================================================
// 5. SCOPE CHAIN
// =============================================================================

setTimeout(() => {
    console.log('5. SCOPE CHAIN:\n');
    
    const globalRate = 0.03; // Global scope
    
    function calculateInterest(balance) {
        const accountType = 'SAVINGS'; // Function scope
        
        function applyBonus() {
            const bonusRate = 0.005; // Nested function scope
            
            // Can access all outer scopes
            const interest = balance * (globalRate + bonusRate);
            
            console.log(`  Nested function accessing:`);
            console.log(`    - Local: bonusRate = ${bonusRate}`);
            console.log(`    - Parent: accountType = ${accountType}`);
            console.log(`    - Parent: balance = ${balance}`);
            console.log(`    - Global: globalRate = ${globalRate}`);
            console.log(`    - Calculated interest: ${interest}\n`);
            
            return interest;
        }
        
        return applyBonus();
    }
    
    calculateInterest(100000);
    
    console.log('  Scope Chain Order:');
    console.log('    1. Local scope (current function)');
    console.log('    2. Outer function scope');
    console.log('    3. Global scope\n');
}, 160);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 180);

// =============================================================================
// 6. PRACTICAL BANKING EXAMPLES
// =============================================================================

setTimeout(() => {
    console.log('6. PRACTICAL BANKING EXAMPLES:\n');
    
    // Example 1: Transaction processor with proper scope
    class TransactionProcessor {
        constructor() {
            this.transactionCount = 0; // Instance scope
        }
        
        processTransaction(amount) {
            const transactionId = `TXN-${++this.transactionCount}`; // Function scope
            
            // Validation in nested scope
            if (amount <= 0) {
                const error = 'Invalid amount'; // Block scope
                console.log(`  ${transactionId}: ${error}`);
                return false;
            }
            
            // Process transaction
            console.log(`  ${transactionId}: Processing ${amount}`);
            return true;
        }
        
        getCount() {
            return this.transactionCount;
        }
    }
    
    const processor = new TransactionProcessor();
    processor.processTransaction(5000);
    processor.processTransaction(-100);
    processor.processTransaction(3000);
    console.log(`  Total transactions: ${processor.getCount()}\n`);
    
    // Example 2: Account factory with closure
    function createBankAccount(initialBalance) {
        let balance = initialBalance; // Private variable (closure)
        const transactions = []; // Private variable (closure)
        
        return {
            deposit(amount) {
                if (amount > 0) {
                    balance += amount;
                    transactions.push({ type: 'DEPOSIT', amount });
                    return balance;
                }
                throw new Error('Invalid amount');
            },
            
            withdraw(amount) {
                if (amount > 0 && amount <= balance) {
                    balance -= amount;
                    transactions.push({ type: 'WITHDRAWAL', amount });
                    return balance;
                }
                throw new Error('Invalid amount or insufficient funds');
            },
            
            getBalance() {
                return balance;
            },
            
            getTransactionCount() {
                return transactions.length;
            }
        };
    }
    
    console.log('Account with closure (private variables):');
    const myAccount = createBankAccount(10000);
    myAccount.deposit(5000);
    myAccount.withdraw(2000);
    console.log(`  Balance: ${myAccount.getBalance()}`);
    console.log(`  Transactions: ${myAccount.getTransactionCount()}`);
    console.log('  ✓ balance and transactions are private\n');
}, 200);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 220);

// =============================================================================
// 7. COMMON SCOPE PITFALLS
// =============================================================================

setTimeout(() => {
    console.log('7. COMMON SCOPE PITFALLS:\n');
    
    console.log('Pitfall 1: Variable shadowing:\n');
    
    const accountBalance = 50000; // Outer scope
    
    function checkBalance() {
        const accountBalance = 30000; // Shadows outer variable
        console.log(`  Inner balance: ${accountBalance}`);
    }
    
    checkBalance();
    console.log(`  Outer balance: ${accountBalance}`);
    console.log('  ⚠️  Inner variable shadows outer variable\n');
    
    console.log('Pitfall 2: Accidental global:\n');
    
    function createTransaction() {
        // Missing var/let/const creates global variable!
        // transactionFee = 10; // DON'T DO THIS
        console.log('  ⚠️  Assigning without declaration creates global variable\n');
    }
    
    createTransaction();
    
    console.log('Pitfall 3: Loop closures with var:\n');
    
    const callbacks = [];
    
    for (var k = 0; k < 3; k++) {
        callbacks.push(() => console.log(`  Callback with var: ${k}`));
    }
    
    console.log('Executing callbacks (all show same value):');
    callbacks.forEach(cb => cb());
    
    console.log('\nFixed with let:');
    const fixedCallbacks = [];
    
    for (let m = 0; m < 3; m++) {
        fixedCallbacks.push(() => console.log(`  Callback with let: ${m}`));
    }
    
    fixedCallbacks.forEach(cb => cb());
    console.log();
}, 240);

setTimeout(() => {
    console.log('='.repeat(70) + '\n');
}, 260);

// =============================================================================
// SCOPE SUMMARY
// =============================================================================

setTimeout(() => {
    console.log('SCOPE SUMMARY:\n');
    
    console.log('Scope Types:');
    console.log('  1. Global Scope');
    console.log('     • Accessible everywhere');
    console.log('     • Avoid polluting global scope');
    console.log('     • Use modules to contain code\n');
    
    console.log('  2. Function Scope (var)');
    console.log('     • Variables accessible throughout function');
    console.log('     • Not block-scoped');
    console.log('     • Hoisted to top of function\n');
    
    console.log('  3. Block Scope (let/const)');
    console.log('     • Variables accessible within {} block');
    console.log('     • Not accessible outside block');
    console.log('     • Preferred over var\n');
    
    console.log('  4. Lexical Scope');
    console.log('     • Inner functions access outer scope');
    console.log('     • Forms closures');
    console.log('     • Enables data privacy\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Use let/const instead of var');
    console.log('  ✓ Minimize global variables');
    console.log('  ✓ Use IIFE or modules for encapsulation');
    console.log('  ✓ Understand scope chain for debugging');
    console.log('  ✓ Be aware of variable shadowing');
    console.log('  ✓ Use closures for data privacy');
    
    console.log();
}, 280);
```

---

## Question 39: Explain the `this` keyword, call/apply/bind, and execution context

### Answer:

The **`this` keyword** refers to the object that is executing the current function. Its value is determined by **how** a function is called, not where it's defined.

### `this` Binding Rules:
1. **Default**: Global object (window/global) or undefined in strict mode
2. **Implicit**: Object that calls the method
3. **Explicit**: call(), apply(), bind()
4. **new**: Newly created object
5. **Arrow functions**: Lexical `this` from enclosing scope

### Banking Scenario: Context Management at Emirates NBD

```javascript
console.log('=== this Keyword & Execution Context - Emirates NBD ===\n');

// =============================================================================
// 1. DEFAULT BINDING
// =============================================================================

console.log('1. DEFAULT BINDING:\n');

function showBankName() {
    console.log(`  this in regular function: ${this === undefined ? 'undefined (strict mode)' : this}`);
}

// In non-strict mode, `this` refers to global object
// In strict mode, `this` is undefined
showBankName();
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. IMPLICIT BINDING (METHOD CALL)
// =============================================================================

console.log('2. IMPLICIT BINDING:\n');

const account = {
    accountNumber: 'ACC-123456789',
    balance: 50000,
    holder: 'Ahmed Al Maktoum',
    
    // Method - `this` refers to the object
    getBalance: function() {
        console.log(`  this.accountNumber: ${this.accountNumber}`);
        console.log(`  this.balance: ${this.balance}`);
        return this.balance;
    },
    
    deposit: function(amount) {
        this.balance += amount;
        console.log(`  Deposited ${amount}. New balance: ${this.balance}`);
        return this.balance;
    }
};

console.log('Calling method on object:');
account.getBalance();
account.deposit(5000);
console.log();

// Pitfall: Losing context
console.log('Losing context:');
const getBalanceFunc = account.getBalance;
// getBalanceFunc(); // `this` is undefined in strict mode
console.log('  ⚠️  When assigning method to variable, context is lost\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. EXPLICIT BINDING (CALL/APPLY/BIND)
// =============================================================================

console.log('3. EXPLICIT BINDING:\n');

const account1 = {
    accountNumber: 'ACC-111111111',
    balance: 30000,
    holder: 'Ali Hassan'
};

const account2 = {
    accountNumber: 'ACC-222222222',
    balance: 70000,
    holder: 'Sara Ahmed'
};

function displayAccountInfo(prefix, suffix) {
    console.log(`  ${prefix} Account: ${this.accountNumber}`);
    console.log(`  Holder: ${this.holder}`);
    console.log(`  Balance: ${this.balance} ${suffix}`);
}

// call() - invoke function with specific `this`, arguments passed individually
console.log('Using call():');
displayAccountInfo.call(account1, '→', 'AED');
console.log();

// apply() - invoke function with specific `this`, arguments as array
console.log('Using apply():');
displayAccountInfo.apply(account2, ['→', 'AED']);
console.log();

// bind() - create new function with specific `this`
console.log('Using bind():');
const displayAccount1Info = displayAccountInfo.bind(account1, '→', 'AED');
displayAccount1Info();
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. NEW BINDING (CONSTRUCTOR)
// =============================================================================

console.log('4. NEW BINDING:\n');

function BankAccount(accountNumber, initialBalance) {
    // `this` refers to the newly created object
    this.accountNumber = accountNumber;
    this.balance = initialBalance;
    this.transactions = [];
    
    console.log(`  Created account: ${this.accountNumber}`);
}

BankAccount.prototype.deposit = function(amount) {
    this.balance += amount;
    this.transactions.push({ type: 'DEPOSIT', amount });
    return this.balance;
};

const newAccount = new BankAccount('ACC-333333333', 40000);
newAccount.deposit(10000);
console.log(`  Balance: ${newAccount.balance}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. ARROW FUNCTIONS (LEXICAL THIS)
// =============================================================================

console.log('5. ARROW FUNCTIONS:\n');

const accountWithArrow = {
    accountNumber: 'ACC-444444444',
    balance: 60000,
    
    // Regular function - has its own `this`
    processTransactions: function(transactions) {
        console.log(`  Processing for: ${this.accountNumber}`);
        
        // Arrow function - inherits `this` from enclosing scope
        transactions.forEach(txn => {
            console.log(`    ${txn.type}: ${txn.amount} (this.accountNumber: ${this.accountNumber})`);
            
            if (txn.type === 'DEPOSIT') {
                this.balance += txn.amount;
            } else {
                this.balance -= txn.amount;
            }
        });
        
        console.log(`  Final balance: ${this.balance}\n`);
    }
};

accountWithArrow.processTransactions([
    { type: 'DEPOSIT', amount: 5000 },
    { type: 'WITHDRAWAL', amount: 2000 }
]);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. PRACTICAL BANKING EXAMPLES
// =============================================================================

console.log('6. PRACTICAL BANKING EXAMPLES:\n');

// Example 1: Event handlers
console.log('Example 1: Event Handler Context\n');

class AccountDashboard {
    constructor(accountNumber) {
        this.accountNumber = accountNumber;
        this.balance = 50000;
    }
    
    // Regular method
    updateBalance(amount) {
        this.balance += amount;
        console.log(`  Updated balance: ${this.balance}`);
    }
    
    // Simulate button click handler
    setupEventHandler() {
        // ❌ Problem: `this` context lost
        // button.onclick = this.updateBalance; // Wrong context
        
        // ✅ Solution 1: bind()
        const handler1 = this.updateBalance.bind(this);
        console.log('  Using bind():');
        handler1(5000);
        
        // ✅ Solution 2: Arrow function
        const handler2 = (amount) => this.updateBalance(amount);
        console.log('  Using arrow function:');
        handler2(3000);
        
        // ✅ Solution 3: Wrapper function
        const handler3 = (amount) => {
            this.updateBalance(amount);
        };
        console.log('  Using wrapper:');
        handler3(2000);
    }
}

const dashboard = new AccountDashboard('ACC-555555555');
dashboard.setupEventHandler();
console.log();

// Example 2: Transaction service
console.log('Example 2: Transaction Service\n');

class TransactionService {
    constructor(serviceName) {
        this.serviceName = serviceName;
        this.processedCount = 0;
    }
    
    processTransaction(transaction) {
        this.processedCount++;
        console.log(`  [${this.serviceName}] Processing transaction #${this.processedCount}`);
        console.log(`    Type: ${transaction.type}, Amount: ${transaction.amount}`);
    }
    
    processBatch(transactions) {
        console.log(`  [${this.serviceName}] Processing batch of ${transactions.length}`);
        
        // Using arrow function to preserve `this`
        transactions.forEach(txn => {
            this.processTransaction(txn);
        });
        
        console.log(`  Total processed: ${this.processedCount}\n`);
    }
}

const service = new TransactionService('Emirates NBD Service');
service.processBatch([
    { type: 'DEPOSIT', amount: 5000 },
    { type: 'WITHDRAWAL', amount: 2000 }
]);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 7. COMMON THIS PITFALLS
// =============================================================================

console.log('7. COMMON THIS PITFALLS:\n');

console.log('Pitfall 1: Method extraction:\n');

const accountObj = {
    accountNumber: 'ACC-666666666',
    balance: 80000,
    
    getInfo: function() {
        return `Account ${this.accountNumber}: ${this.balance}`;
    }
};

// ❌ Context lost
const getInfo = accountObj.getInfo;
// console.log(getInfo()); // Error: cannot read accountNumber of undefined

// ✅ Fixed with bind
const getInfoBound = accountObj.getInfo.bind(accountObj);
console.log(`  ${getInfoBound()}\n`);

console.log('Pitfall 2: Callbacks:\n');

const accountObj2 = {
    accountNumber: 'ACC-777777777',
    transactions: [5000, 3000, 2000],
    
    sumTransactions: function() {
        // ❌ Problem: `this` in callback
        // return this.transactions.reduce(function(sum, amount) {
        //     console.log(this); // undefined or global object
        //     return sum + amount;
        // }, 0);
        
        // ✅ Solution: Arrow function
        return this.transactions.reduce((sum, amount) => {
            return sum + amount;
        }, 0);
    }
};

console.log(`  Total transactions: ${accountObj2.sumTransactions()}\n`);

console.log('Pitfall 3: Nested functions:\n');

const accountObj3 = {
    accountNumber: 'ACC-888888888',
    
    processAccount: function() {
        console.log(`  Outer: ${this.accountNumber}`);
        
        function innerFunction() {
            // ❌ `this` is not the same as outer function
            // console.log(`  Inner: ${this.accountNumber}`); // undefined
        }
        
        // ✅ Solution 1: Save reference
        const self = this;
        function innerFixed1() {
            console.log(`  Inner (self): ${self.accountNumber}`);
        }
        innerFixed1();
        
        // ✅ Solution 2: Arrow function
        const innerFixed2 = () => {
            console.log(`  Inner (arrow): ${this.accountNumber}`);
        };
        innerFixed2();
    }
};

accountObj3.processAccount();
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// THIS KEYWORD SUMMARY
// =============================================================================

console.log('THIS KEYWORD SUMMARY:\n');

console.log('Binding Rules (Priority Order):');
console.log('  1. new binding: `this` = newly created object');
console.log('  2. Explicit binding: call/apply/bind');
console.log('  3. Implicit binding: object.method()');
console.log('  4. Default binding: global or undefined\n');

console.log('call/apply/bind:');
console.log('  • call(thisArg, arg1, arg2, ...)');
console.log('    - Invokes function immediately');
console.log('    - Arguments passed individually');
console.log('  • apply(thisArg, [argsArray])');
console.log('    - Invokes function immediately');
console.log('    - Arguments as array');
console.log('  • bind(thisArg, arg1, arg2, ...)');
console.log('    - Returns new function');
console.log('    - Does not invoke immediately');
console.log('    - Can partially apply arguments\n');

console.log('Arrow Functions:');
console.log('  • Do NOT have their own `this`');
console.log('  • Inherit `this` from enclosing scope (lexical)');
console.log('  • Cannot be bound with call/apply/bind');
console.log('  • Cannot be used as constructors\n');

console.log('Best Practices:');
console.log('  ✓ Use arrow functions for callbacks');
console.log('  ✓ Use bind() when passing methods as callbacks');
console.log('  ✓ Use const self = this pattern for nested functions');
console.log('  ✓ Understand context when debugging');
console.log('  ✓ Avoid complex `this` logic when possible');
console.log('  ✓ Consider using closures for data privacy');

console.log();
```

### Key Takeaways:

**Data Types**:
- 7 primitive types + 1 reference type (object)
- Primitives stored by value, objects by reference
- Always use === for comparisons (no type coercion)
- Explicit type conversion is safer than implicit coercion

**Scope**:
- Global, function (var), block (let/const), lexical
- Use let/const instead of var
- Closures enable data privacy
- Understand scope chain for debugging

**this Keyword**:
- Determined by how function is called
- call/apply for immediate invocation
- bind for creating new function
- Arrow functions inherit lexical `this`
- Common pitfall: losing context in callbacks

---

**End of Questions 37-39**
