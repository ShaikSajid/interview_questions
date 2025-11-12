# Questions 34-36: TypedArrays, BigInt & Internationalization

## Question 34: What are TypedArrays in JavaScript? When and why would you use them?

### Answer:

**TypedArrays** are array-like objects that provide a mechanism for reading and writing raw binary data in memory buffers. They offer better performance for numeric computations and are essential for working with binary data.

### Types of TypedArrays:
- **Int8Array, Uint8Array**: 8-bit integers
- **Int16Array, Uint16Array**: 16-bit integers
- **Int32Array, Uint32Array**: 32-bit integers
- **Float32Array, Float64Array**: Floating-point numbers
- **BigInt64Array, BigUint64Array**: 64-bit integers

### Banking Scenario: High-Performance Data Processing at Emirates NBD

```javascript
console.log('=== TypedArrays Demo - Emirates NBD ===\n');

// =============================================================================
// 1. TYPEDARRAYS BASICS
// =============================================================================

console.log('1. TYPEDARRAYS BASICS:\n');

// Creating TypedArrays
const int32Array = new Int32Array(5);
const float64Array = new Float64Array(5);
const uint8Array = new Uint8Array([255, 128, 64, 32, 16]);

console.log('TypedArray Creation:');
console.log(`  Int32Array: ${int32Array}`);
console.log(`  Float64Array: ${float64Array}`);
console.log(`  Uint8Array: ${uint8Array}`);
console.log(`  Length: ${uint8Array.length}`);
console.log(`  Byte length: ${uint8Array.byteLength}`);
console.log(`  Bytes per element: ${uint8Array.BYTES_PER_ELEMENT}\n`);

// ArrayBuffer - underlying binary data
const buffer = new ArrayBuffer(16); // 16 bytes
const view32 = new Int32Array(buffer); // 4 elements (4 bytes each)
const view8 = new Uint8Array(buffer);   // 16 elements (1 byte each)

view32[0] = 0x12345678;

console.log('Shared ArrayBuffer:');
console.log(`  Buffer byte length: ${buffer.byteLength}`);
console.log(`  Int32Array view: [${view32}]`);
console.log(`  Uint8Array view: [${view8}]`);
console.log('  Same buffer, different views\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. PERFORMANCE COMPARISON
// =============================================================================

console.log('2. PERFORMANCE COMPARISON:\n');

class PerformanceTest {
    static compareArrayPerformance(size = 1000000) {
        console.log(`Processing ${size.toLocaleString()} transaction amounts:\n`);
        
        // Regular Array
        const startRegular = performance.now();
        const regularArray = new Array(size);
        for (let i = 0; i < size; i++) {
            regularArray[i] = Math.random() * 100000;
        }
        const sumRegular = regularArray.reduce((a, b) => a + b, 0);
        const endRegular = performance.now();
        
        console.log('Regular Array:');
        console.log(`  Time: ${(endRegular - startRegular).toFixed(2)}ms`);
        console.log(`  Sum: AED ${sumRegular.toFixed(2)}`);
        console.log(`  Memory: ~${(size * 8).toLocaleString()} bytes (estimated)\n`);
        
        // TypedArray
        const startTyped = performance.now();
        const typedArray = new Float64Array(size);
        for (let i = 0; i < size; i++) {
            typedArray[i] = Math.random() * 100000;
        }
        const sumTyped = typedArray.reduce((a, b) => a + b, 0);
        const endTyped = performance.now();
        
        console.log('Float64Array:');
        console.log(`  Time: ${(endTyped - startTyped).toFixed(2)}ms`);
        console.log(`  Sum: AED ${sumTyped.toFixed(2)}`);
        console.log(`  Memory: ${typedArray.byteLength.toLocaleString()} bytes\n`);
        
        const improvement = ((endRegular - startRegular) / (endTyped - startTyped) - 1) * 100;
        console.log(`Performance improvement: ${improvement.toFixed(1)}%\n`);
    }
}

PerformanceTest.compareArrayPerformance(100000);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. BANKING USE CASES
// =============================================================================

console.log('3. BANKING USE CASES:\n');

// Use Case 1: High-frequency transaction processing
class TransactionProcessor {
    constructor(bufferSize = 10000) {
        // Store transaction amounts in TypedArray for performance
        this.amounts = new Float64Array(bufferSize);
        this.timestamps = new BigUint64Array(bufferSize);
        this.accountIds = new Uint32Array(bufferSize);
        this.currentIndex = 0;
    }
    
    addTransaction(accountId, amount) {
        if (this.currentIndex >= this.amounts.length) {
            throw new Error('Buffer full');
        }
        
        this.accountIds[this.currentIndex] = accountId;
        this.amounts[this.currentIndex] = amount;
        this.timestamps[this.currentIndex] = BigInt(Date.now());
        
        this.currentIndex++;
    }
    
    getTotalVolume() {
        return this.amounts.slice(0, this.currentIndex).reduce((a, b) => a + b, 0);
    }
    
    getAverageTransaction() {
        if (this.currentIndex === 0) return 0;
        return this.getTotalVolume() / this.currentIndex;
    }
    
    getTransactionsByAccount(accountId) {
        const transactions = [];
        for (let i = 0; i < this.currentIndex; i++) {
            if (this.accountIds[i] === accountId) {
                transactions.push({
                    accountId: this.accountIds[i],
                    amount: this.amounts[i],
                    timestamp: Number(this.timestamps[i])
                });
            }
        }
        return transactions;
    }
    
    getStatistics() {
        const slice = this.amounts.slice(0, this.currentIndex);
        return {
            count: this.currentIndex,
            total: this.getTotalVolume(),
            average: this.getAverageTransaction(),
            min: Math.min(...slice),
            max: Math.max(...slice)
        };
    }
}

const processor = new TransactionProcessor(1000);

// Add sample transactions
console.log('Adding transactions...\n');
for (let i = 0; i < 100; i++) {
    const accountId = Math.floor(Math.random() * 10) + 123456789;
    const amount = Math.random() * 50000 + 1000;
    processor.addTransaction(accountId, amount);
}

const stats = processor.getStatistics();
console.log('Transaction Statistics:');
console.log(`  Count: ${stats.count}`);
console.log(`  Total volume: AED ${stats.total.toFixed(2)}`);
console.log(`  Average: AED ${stats.average.toFixed(2)}`);
console.log(`  Min: AED ${stats.min.toFixed(2)}`);
console.log(`  Max: AED ${stats.max.toFixed(2)}\n`);

// Use Case 2: Binary data encoding/decoding
class BinaryTransactionEncoder {
    // Encode transaction to binary format
    static encode(transaction) {
        // Layout: [accountId(4) | amount(8) | timestamp(8) | type(1)]
        const buffer = new ArrayBuffer(21);
        const view = new DataView(buffer);
        
        // Account ID (4 bytes)
        view.setUint32(0, transaction.accountId, true);
        
        // Amount (8 bytes, as float64)
        view.setFloat64(4, transaction.amount, true);
        
        // Timestamp (8 bytes, as BigInt)
        view.setBigUint64(12, BigInt(transaction.timestamp), true);
        
        // Type (1 byte: 1=DEPOSIT, 2=WITHDRAWAL, 3=TRANSFER)
        view.setUint8(20, transaction.type);
        
        return buffer;
    }
    
    // Decode binary to transaction object
    static decode(buffer) {
        const view = new DataView(buffer);
        
        return {
            accountId: view.getUint32(0, true),
            amount: view.getFloat64(4, true),
            timestamp: Number(view.getBigUint64(12, true)),
            type: view.getUint8(20)
        };
    }
    
    static getTypeName(typeCode) {
        const types = { 1: 'DEPOSIT', 2: 'WITHDRAWAL', 3: 'TRANSFER' };
        return types[typeCode] || 'UNKNOWN';
    }
}

console.log('Binary Transaction Encoding:\n');

const transaction = {
    accountId: 123456789,
    amount: 5000.50,
    timestamp: Date.now(),
    type: 1
};

console.log('Original transaction:');
console.log(`  Account: ${transaction.accountId}`);
console.log(`  Amount: AED ${transaction.amount}`);
console.log(`  Timestamp: ${new Date(transaction.timestamp).toISOString()}`);
console.log(`  Type: ${BinaryTransactionEncoder.getTypeName(transaction.type)}\n`);

const encoded = BinaryTransactionEncoder.encode(transaction);
console.log('Encoded:');
console.log(`  Byte length: ${encoded.byteLength} bytes`);
console.log(`  Hex: ${Array.from(new Uint8Array(encoded))
    .map(b => b.toString(16).padStart(2, '0'))
    .join(' ')}\n`);

const decoded = BinaryTransactionEncoder.decode(encoded);
console.log('Decoded transaction:');
console.log(`  Account: ${decoded.accountId}`);
console.log(`  Amount: AED ${decoded.amount}`);
console.log(`  Timestamp: ${new Date(decoded.timestamp).toISOString()}`);
console.log(`  Type: ${BinaryTransactionEncoder.getTypeName(decoded.type)}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// TYPEDARRAYS SUMMARY
// =============================================================================

console.log('TYPEDARRAYS SUMMARY:\n');

console.log('Available TypedArrays:');
console.log('  Int8Array     - Signed 8-bit integer');
console.log('  Uint8Array    - Unsigned 8-bit integer');
console.log('  Int16Array    - Signed 16-bit integer');
console.log('  Uint16Array   - Unsigned 16-bit integer');
console.log('  Int32Array    - Signed 32-bit integer');
console.log('  Uint32Array   - Unsigned 32-bit integer');
console.log('  Float32Array  - 32-bit floating point');
console.log('  Float64Array  - 64-bit floating point');
console.log('  BigInt64Array - Signed 64-bit integer');
console.log('  BigUint64Array- Unsigned 64-bit integer');

console.log('\nAdvantages:');
console.log('  ✓ Better performance for numeric operations');
console.log('  ✓ Predictable memory layout');
console.log('  ✓ Lower memory overhead');
console.log('  ✓ Direct binary data manipulation');
console.log('  ✓ Ideal for large datasets');

console.log('\nUse Cases:');
console.log('  • High-frequency trading data');
console.log('  • Binary protocol implementation');
console.log('  • Image/audio processing');
console.log('  • WebGL graphics');
console.log('  • Network data transmission');
console.log('  • File format parsing');

console.log();
```

---

## Question 35: What is BigInt in JavaScript? When should you use it?

### Answer:

**BigInt** is a built-in object that provides a way to represent whole numbers larger than 2^53 - 1 (Number.MAX_SAFE_INTEGER). It's essential for precise calculations with very large integers.

### Key Features:
- Can represent arbitrarily large integers
- Suffix with 'n' or use BigInt() constructor
- Cannot be mixed with regular numbers in operations
- No decimal points (integers only)

### Banking Scenario: Large Financial Calculations at Emirates NBD

```javascript
console.log('=== BigInt Demo - Emirates NBD ===\n');

// =============================================================================
// 1. BIGINT BASICS
// =============================================================================

console.log('1. BIGINT BASICS:\n');

// Number limitations
console.log('Number limitations:');
console.log(`  MAX_SAFE_INTEGER: ${Number.MAX_SAFE_INTEGER}`);
console.log(`  MAX_SAFE_INTEGER + 1: ${Number.MAX_SAFE_INTEGER + 1}`);
console.log(`  MAX_SAFE_INTEGER + 2: ${Number.MAX_SAFE_INTEGER + 2}`);
console.log('  ⚠️  Precision lost!\n');

// BigInt creation
const bigInt1 = 123456789012345678901234567890n;
const bigInt2 = BigInt('123456789012345678901234567890');
const bigInt3 = BigInt(123456789);

console.log('BigInt creation:');
console.log(`  Literal (n suffix): ${bigInt1}`);
console.log(`  Constructor (string): ${bigInt2}`);
console.log(`  Constructor (number): ${bigInt3}\n`);

// BigInt operations
console.log('BigInt operations:');
const a = 1000000000000000000n;
const b = 2000000000000000000n;

console.log(`  Addition: ${a} + ${b} = ${a + b}`);
console.log(`  Subtraction: ${b} - ${a} = ${b - a}`);
console.log(`  Multiplication: ${a} * 2n = ${a * 2n}`);
console.log(`  Division: ${b} / ${a} = ${b / a}`);
console.log(`  Modulo: ${b} % 300n = ${b % 300n}`);
console.log(`  Exponentiation: 10n ** 20n = ${10n ** 20n}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. BANKING USE CASES
// =============================================================================

console.log('2. BANKING USE CASES:\n');

// Use Case 1: Precise financial calculations (amounts in fils/cents)
class PreciseAccount {
    // Store amounts in fils (1 AED = 100 fils) as BigInt for precision
    constructor(accountNumber, balanceAED = 0) {
        this.accountNumber = accountNumber;
        this.balanceFils = BigInt(Math.round(balanceAED * 100));
    }
    
    getBalanceAED() {
        return Number(this.balanceFils) / 100;
    }
    
    depositAED(amount) {
        const fils = BigInt(Math.round(amount * 100));
        this.balanceFils += fils;
        return this.getBalanceAED();
    }
    
    withdrawAED(amount) {
        const fils = BigInt(Math.round(amount * 100));
        
        if (fils > this.balanceFils) {
            throw new Error('Insufficient funds');
        }
        
        this.balanceFils -= fils;
        return this.getBalanceAED();
    }
    
    calculateInterest(annualRate, days) {
        // Interest = (balance * rate * days) / (365 * 100)
        const rate = BigInt(Math.round(annualRate * 100));
        const daysBig = BigInt(days);
        
        const interest = (this.balanceFils * rate * daysBig) / (36500n * 100n);
        
        return Number(interest) / 100;
    }
    
    addInterest(annualRate, days) {
        const interestAED = this.calculateInterest(annualRate, days);
        const interestFils = BigInt(Math.round(interestAED * 100));
        this.balanceFils += interestFils;
        return this.getBalanceAED();
    }
}

console.log('Precise Financial Calculations:\n');

const account = new PreciseAccount('ACC-123456789', 50000.00);

console.log('Initial balance:', `AED ${account.getBalanceAED()}`);

account.depositAED(0.01); // 1 fils
console.log('After depositing 1 fils:', `AED ${account.getBalanceAED()}`);

account.depositAED(0.05); // 5 fils
console.log('After depositing 5 fils:', `AED ${account.getBalanceAED()}`);

const interest = account.calculateInterest(3.5, 30);
console.log(`\nInterest calculation (3.5% for 30 days): AED ${interest.toFixed(2)}`);

account.addInterest(3.5, 30);
console.log('Balance after interest:', `AED ${account.getBalanceAED().toFixed(2)}\n`);

// Use Case 2: Large account numbers and IDs
class LargeNumberOperations {
    // Generate unique transaction ID using timestamp + counter
    static transactionCounter = 0n;
    
    static generateTransactionId() {
        const timestamp = BigInt(Date.now());
        const counter = this.transactionCounter++;
        
        // Combine: timestamp (milliseconds) * 1000000 + counter
        return timestamp * 1000000n + counter;
    }
    
    // Calculate total transaction volume (in fils)
    static calculateTotalVolume(transactions) {
        let total = 0n;
        
        for (const txn of transactions) {
            total += txn.amountFils;
        }
        
        return total;
    }
    
    // Format large numbers with separators
    static formatBigNumber(value) {
        const str = value.toString();
        return str.replace(/\B(?=(\d{3})+(?!\d))/g, ',');
    }
}

console.log('Large Number Operations:\n');

// Generate unique transaction IDs
console.log('Generated transaction IDs:');
for (let i = 0; i < 5; i++) {
    const txnId = LargeNumberOperations.generateTransactionId();
    console.log(`  ${i + 1}. ${txnId}`);
}
console.log();

// Calculate large volumes
const largeTransactions = [
    { id: 1, amountFils: 5000000000n },  // 50,000,000 AED
    { id: 2, amountFils: 7500000000n },  // 75,000,000 AED
    { id: 3, amountFils: 12000000000n }, // 120,000,000 AED
    { id: 4, amountFils: 3500000000n }   // 35,000,000 AED
];

const totalFils = LargeNumberOperations.calculateTotalVolume(largeTransactions);
const totalAED = Number(totalFils) / 100;

console.log('Large volume calculation:');
console.log(`  Total (fils): ${LargeNumberOperations.formatBigNumber(totalFils)}`);
console.log(`  Total (AED): ${LargeNumberOperations.formatBigNumber(BigInt(totalAED))}\n`);

// Use Case 3: Cryptocurrency and blockchain amounts
class CryptoWallet {
    // Bitcoin uses satoshis (1 BTC = 100,000,000 satoshis)
    constructor(walletAddress) {
        this.walletAddress = walletAddress;
        this.balanceSatoshis = 0n;
    }
    
    depositBTC(btc) {
        const satoshis = BigInt(Math.round(btc * 100000000));
        this.balanceSatoshis += satoshis;
    }
    
    getBalanceBTC() {
        return Number(this.balanceSatoshis) / 100000000;
    }
    
    convertToAED(btcToAEDRate) {
        const btc = this.getBalanceBTC();
        return btc * btcToAEDRate;
    }
    
    calculateGasFee(gasPrice, gasUnits) {
        // Ethereum gas calculations (wei)
        return BigInt(gasPrice) * BigInt(gasUnits);
    }
}

console.log('Cryptocurrency Operations:\n');

const wallet = new CryptoWallet('0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb');

wallet.depositBTC(0.05);
console.log(`Bitcoin balance: ${wallet.getBalanceBTC()} BTC`);
console.log(`  Satoshis: ${wallet.balanceSatoshis}`);

const btcToAED = 150000;
console.log(`  Value in AED: ${wallet.convertToAED(btcToAED).toFixed(2)} (@ ${btcToAED} AED/BTC)\n`);

const gasFee = wallet.calculateGasFee(50, 21000);
console.log(`Gas fee calculation:`);
console.log(`  Gas price: 50 gwei`);
console.log(`  Gas units: 21,000`);
console.log(`  Total fee: ${gasFee} wei\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. BIGINT VS NUMBER COMPARISON
// =============================================================================

console.log('3. BIGINT VS NUMBER COMPARISON:\n');

class BigIntVsNumber {
    static demonstratePrecision() {
        console.log('Precision comparison:\n');
        
        // Number loses precision
        console.log('Using Number:');
        let numSum = 0;
        for (let i = 0; i < 1000000; i++) {
            numSum += 0.001;
        }
        console.log(`  0.001 × 1,000,000 = ${numSum}`);
        console.log(`  Expected: 1000`);
        console.log(`  Error: ${Math.abs(1000 - numSum).toFixed(10)}\n`);
        
        // BigInt maintains precision (using fils)
        console.log('Using BigInt (in fils):');
        let bigSum = 0n;
        for (let i = 0; i < 1000000; i++) {
            bigSum += 100n; // 0.001 AED = 0.1 fils
        }
        const result = Number(bigSum) / 100;
        console.log(`  0.1 fils × 1,000,000 = ${result} AED`);
        console.log(`  Expected: 1000`);
        console.log(`  Error: 0\n`);
    }
    
    static demonstrateLimits() {
        console.log('Range comparison:\n');
        
        console.log('Number limits:');
        console.log(`  MAX_SAFE_INTEGER: ${Number.MAX_SAFE_INTEGER.toLocaleString()}`);
        console.log(`  Can represent: ±${(Number.MAX_SAFE_INTEGER / 1e15).toFixed(0)} quadrillion\n`);
        
        console.log('BigInt limits:');
        const hugeBigInt = 10n ** 100n;
        console.log(`  Can represent: arbitrarily large`);
        console.log(`  Example: 10^100 = ${hugeBigInt.toString().substring(0, 50)}...\n`);
    }
}

BigIntVsNumber.demonstratePrecision();
BigIntVsNumber.demonstrateLimits();

console.log('='.repeat(70) + '\n');

// =============================================================================
// BIGINT SUMMARY
// =============================================================================

console.log('BIGINT SUMMARY:\n');

console.log('Key Features:');
console.log('  ✓ Arbitrary precision integers');
console.log('  ✓ No upper limit (memory dependent)');
console.log('  ✓ Exact calculations');
console.log('  ✓ No floating-point errors');

console.log('\nCreation:');
console.log('  • Literal: 123n');
console.log('  • Constructor: BigInt("123")');
console.log('  • Cannot mix with Number in operations');

console.log('\nOperations:');
console.log('  ✓ Arithmetic: +, -, *, /, %, **');
console.log('  ✓ Comparison: <, >, <=, >=, ==, ===');
console.log('  ✓ Bitwise: &, |, ^, ~, <<, >>');
console.log('  ✗ Math methods (use custom functions)');

console.log('\nUse Cases:');
console.log('  • Financial calculations (cents/fils)');
console.log('  • Large IDs and counters');
console.log('  • Cryptocurrency amounts');
console.log('  • Timestamp arithmetic');
console.log('  • Database primary keys');
console.log('  • Scientific calculations');

console.log('\nConversions:');
console.log('  • BigInt → Number: Number(bigint)');
console.log('  • Number → BigInt: BigInt(number)');
console.log('  • String → BigInt: BigInt(string)');
console.log('  ⚠️  May lose precision when converting large BigInt to Number');

console.log();
```

---

## Question 36: What is the Internationalization API (Intl) in JavaScript?

### Answer:

The **Intl API** provides language-sensitive string comparison, number formatting, and date/time formatting. It's essential for building applications that support multiple languages and locales.

### Key Objects:
- **Intl.DateTimeFormat**: Date/time formatting
- **Intl.NumberFormat**: Number formatting
- **Intl.Collator**: String comparison
- **Intl.PluralRules**: Pluralization rules
- **Intl.RelativeTimeFormat**: Relative time formatting

### Banking Scenario: Multi-locale Support at Emirates NBD

```javascript
console.log('=== Internationalization API Demo - Emirates NBD ===\n');

// =============================================================================
// 1. NUMBER FORMATTING
// =============================================================================

console.log('1. NUMBER FORMATTING (Intl.NumberFormat):\n');

class CurrencyFormatter {
    // Format amounts in different currencies and locales
    static formatCurrency(amount, currency, locale = 'en-AE') {
        const formatter = new Intl.NumberFormat(locale, {
            style: 'currency',
            currency: currency
        });
        return formatter.format(amount);
    }
    
    // Format with custom options
    static formatWithOptions(amount, options, locale = 'en-AE') {
        const formatter = new Intl.NumberFormat(locale, options);
        return formatter.format(amount);
    }
    
    // Demonstrate various locales
    static demonstrateLocales(amount) {
        const locales = [
            { code: 'en-AE', name: 'English (UAE)', currency: 'AED' },
            { code: 'ar-AE', name: 'Arabic (UAE)', currency: 'AED' },
            { code: 'en-US', name: 'English (US)', currency: 'USD' },
            { code: 'en-GB', name: 'English (UK)', currency: 'GBP' },
            { code: 'fr-FR', name: 'French (France)', currency: 'EUR' },
            { code: 'ja-JP', name: 'Japanese', currency: 'JPY' },
            { code: 'zh-CN', name: 'Chinese', currency: 'CNY' },
            { code: 'hi-IN', name: 'Hindi (India)', currency: 'INR' }
        ];
        
        console.log(`Amount: ${amount}\n`);
        
        locales.forEach(({ code, name, currency }) => {
            const formatted = this.formatCurrency(amount, currency, code);
            console.log(`  ${name.padEnd(20)} ${formatted}`);
        });
        console.log();
    }
}

// Demonstrate currency formatting
console.log('Currency formatting in different locales:\n');
CurrencyFormatter.demonstrateLocales(50000);

// Custom formatting options
console.log('Custom number formatting:\n');

const customFormats = [
    { label: 'Standard', options: { style: 'decimal' } },
    { label: 'With grouping', options: { style: 'decimal', useGrouping: true } },
    { label: '2 decimals', options: { minimumFractionDigits: 2, maximumFractionDigits: 2 } },
    { label: 'Percentage', options: { style: 'percent', minimumFractionDigits: 2 } },
    { label: 'Compact', options: { notation: 'compact' } },
    { label: 'Scientific', options: { notation: 'scientific' } },
    { label: 'Engineering', options: { notation: 'engineering' } }
];

customFormats.forEach(({ label, options }) => {
    const formatted = CurrencyFormatter.formatWithOptions(1234567.89, options);
    console.log(`  ${label.padEnd(20)} ${formatted}`);
});
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. DATE/TIME FORMATTING
// =============================================================================

console.log('2. DATE/TIME FORMATTING (Intl.DateTimeFormat):\n');

class DateFormatter {
    static formatDate(date, locale, options = {}) {
        const formatter = new Intl.DateTimeFormat(locale, options);
        return formatter.format(date);
    }
    
    static demonstrateDateFormats() {
        const now = new Date();
        
        console.log(`Date: ${now.toISOString()}\n`);
        
        const formats = [
            { locale: 'en-AE', name: 'English (UAE)', options: { dateStyle: 'full' } },
            { locale: 'ar-AE', name: 'Arabic (UAE)', options: { dateStyle: 'full' } },
            { locale: 'en-US', name: 'English (US)', options: { dateStyle: 'full' } },
            { locale: 'fr-FR', name: 'French', options: { dateStyle: 'full' } },
            { locale: 'ja-JP', name: 'Japanese', options: { dateStyle: 'full' } }
        ];
        
        console.log('Full date format:\n');
        formats.forEach(({ locale, name, options }) => {
            const formatted = this.formatDate(now, locale, options);
            console.log(`  ${name.padEnd(20)} ${formatted}`);
        });
        console.log();
        
        console.log('Custom date/time formats:\n');
        
        const customFormats = [
            { label: 'Date only', options: { dateStyle: 'short' } },
            { label: 'Time only', options: { timeStyle: 'short' } },
            { label: 'Date + Time', options: { dateStyle: 'medium', timeStyle: 'short' } },
            { label: 'Custom', options: { year: 'numeric', month: 'long', day: 'numeric' } },
            { label: 'Weekday', options: { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' } }
        ];
        
        customFormats.forEach(({ label, options }) => {
            const formatted = this.formatDate(now, 'en-AE', options);
            console.log(`  ${label.padEnd(20)} ${formatted}`);
        });
        console.log();
    }
}

DateFormatter.demonstrateDateFormats();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. RELATIVE TIME FORMATTING
// =============================================================================

console.log('3. RELATIVE TIME FORMATTING (Intl.RelativeTimeFormat):\n');

class RelativeTimeFormatter {
    static format(value, unit, locale = 'en-AE', style = 'long') {
        const formatter = new Intl.RelativeTimeFormat(locale, { style });
        return formatter.format(value, unit);
    }
    
    static demonstrateRelativeTime() {
        console.log('Transaction timestamps:\n');
        
        const times = [
            { value: -1, unit: 'second', label: '1 second ago' },
            { value: -30, unit: 'second', label: '30 seconds ago' },
            { value: -5, unit: 'minute', label: '5 minutes ago' },
            { value: -2, unit: 'hour', label: '2 hours ago' },
            { value: -1, unit: 'day', label: 'Yesterday' },
            { value: -7, unit: 'day', label: 'Last week' },
            { value: 1, unit: 'day', label: 'Tomorrow' },
            { value: 7, unit: 'day', label: 'Next week' }
        ];
        
        const locales = ['en-AE', 'ar-AE', 'fr-FR'];
        
        times.forEach(({ value, unit }) => {
            console.log(`  ${unit} ${value}:`);
            locales.forEach(locale => {
                const formatted = this.format(value, unit, locale);
                console.log(`    ${locale}: ${formatted}`);
            });
            console.log();
        });
    }
}

RelativeTimeFormatter.demonstrateRelativeTime();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. STRING COLLATION
// =============================================================================

console.log('4. STRING COLLATION (Intl.Collator):\n');

class StringCollator {
    static compare(str1, str2, locale = 'en', options = {}) {
        const collator = new Intl.Collator(locale, options);
        return collator.compare(str1, str2);
    }
    
    static sortAccounts(accounts, locale = 'en') {
        const collator = new Intl.Collator(locale, { sensitivity: 'base' });
        return accounts.sort((a, b) => collator.compare(a.name, b.name));
    }
    
    static demonstrateCollation() {
        console.log('Account name sorting:\n');
        
        const accounts = [
            { id: 1, name: 'Zayed Ahmed' },
            { id: 2, name: 'Ahmed Ali' },
            { id: 3, name: 'محمد خالد' },
            { id: 4, name: 'أحمد علي' },
            { id: 5, name: 'Amélie Dubois' },
            { id: 6, name: 'Björk Sigurðsson' }
        ];
        
        console.log('English locale sorting:');
        const sortedEN = this.sortAccounts([...accounts], 'en');
        sortedEN.forEach(acc => console.log(`  ${acc.name}`));
        console.log();
        
        console.log('Arabic locale sorting:');
        const sortedAR = this.sortAccounts([...accounts], 'ar');
        sortedAR.forEach(acc => console.log(`  ${acc.name}`));
        console.log();
        
        // Case-insensitive comparison
        console.log('Case-insensitive comparison:');
        const result = this.compare('Ahmed', 'ahmed', 'en', { sensitivity: 'base' });
        console.log(`  'Ahmed' vs 'ahmed': ${result === 0 ? 'Equal' : 'Different'}\n`);
    }
}

StringCollator.demonstrateCollation();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PRACTICAL BANKING APPLICATION
// =============================================================================

console.log('5. PRACTICAL BANKING APPLICATION:\n');

class MultilingualBankingService {
    constructor(locale = 'en-AE') {
        this.locale = locale;
        this.numberFormatter = new Intl.NumberFormat(locale, {
            style: 'currency',
            currency: locale.startsWith('ar') || locale.startsWith('en-AE') ? 'AED' : 'USD'
        });
        this.dateFormatter = new Intl.DateTimeFormat(locale, {
            dateStyle: 'medium',
            timeStyle: 'short'
        });
        this.relativeTimeFormatter = new Intl.RelativeTimeFormat(locale, {
            style: 'long'
        });
    }
    
    formatAccountBalance(balance) {
        return this.numberFormatter.format(balance);
    }
    
    formatTransactionDate(date) {
        return this.dateFormatter.format(date);
    }
    
    formatRelativeTime(date) {
        const now = Date.now();
        const diff = date.getTime() - now;
        const diffMinutes = Math.round(diff / 60000);
        const diffHours = Math.round(diff / 3600000);
        const diffDays = Math.round(diff / 86400000);
        
        if (Math.abs(diffMinutes) < 60) {
            return this.relativeTimeFormatter.format(diffMinutes, 'minute');
        } else if (Math.abs(diffHours) < 24) {
            return this.relativeTimeFormatter.format(diffHours, 'hour');
        } else {
            return this.relativeTimeFormatter.format(diffDays, 'day');
        }
    }
    
    generateAccountStatement(account, transactions) {
        const statement = [];
        
        statement.push('='.repeat(60));
        statement.push(`Account Statement - ${this.locale.toUpperCase()}`);
        statement.push('='.repeat(60));
        statement.push('');
        statement.push(`Account: ${account.accountNumber}`);
        statement.push(`Balance: ${this.formatAccountBalance(account.balance)}`);
        statement.push(`Generated: ${this.formatTransactionDate(new Date())}`);
        statement.push('');
        statement.push('Recent Transactions:');
        statement.push('-'.repeat(60));
        
        transactions.forEach((txn, i) => {
            const amount = this.numberFormatter.format(txn.amount);
            const date = this.formatTransactionDate(txn.date);
            const relative = this.formatRelativeTime(txn.date);
            statement.push(`${i + 1}. ${txn.type.padEnd(12)} ${amount.padEnd(15)} ${date}`);
            statement.push(`   ${relative}`);
        });
        
        statement.push('='.repeat(60));
        
        return statement.join('\n');
    }
}

// Demonstrate multilingual statements
const account = {
    accountNumber: 'ACC-123456789',
    balance: 75000
};

const transactions = [
    { type: 'DEPOSIT', amount: 5000, date: new Date(Date.now() - 3600000) },
    { type: 'WITHDRAWAL', amount: 2000, date: new Date(Date.now() - 86400000) },
    { type: 'TRANSFER', amount: 10000, date: new Date(Date.now() - 172800000) }
];

console.log('English (UAE) Statement:\n');
const serviceEN = new MultilingualBankingService('en-AE');
console.log(serviceEN.generateAccountStatement(account, transactions));
console.log('\n');

console.log('Arabic (UAE) Statement:\n');
const serviceAR = new MultilingualBankingService('ar-AE');
console.log(serviceAR.generateAccountStatement(account, transactions));
console.log('\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// INTL API SUMMARY
// =============================================================================

console.log('INTL API SUMMARY:\n');

console.log('Core Objects:');
console.log('  1. Intl.NumberFormat     - Number/currency formatting');
console.log('  2. Intl.DateTimeFormat   - Date/time formatting');
console.log('  3. Intl.RelativeTimeFormat - Relative time ("2 days ago")');
console.log('  4. Intl.Collator         - String comparison/sorting');
console.log('  5. Intl.PluralRules      - Pluralization rules');
console.log('  6. Intl.ListFormat       - List formatting');
console.log('  7. Intl.DisplayNames     - Language/region names');

console.log('\nKey Features:');
console.log('  ✓ Locale-aware formatting');
console.log('  ✓ Currency conversion display');
console.log('  ✓ Date/time localization');
console.log('  ✓ Proper string sorting');
console.log('  ✓ RTL language support');

console.log('\nUse Cases:');
console.log('  • Multi-currency support');
console.log('  • International banking apps');
console.log('  • Localized reports');
console.log('  • Transaction timestamps');
console.log('  • Customer-facing interfaces');
console.log('  • Compliance with local formats');

console.log();
```

### Key Takeaways:

**TypedArrays**:
- Efficient binary data handling
- Better performance for large datasets
- Essential for WebGL, audio, networking
- Various types for different data sizes

**BigInt**:
- Arbitrary precision integers
- Essential for large financial calculations
- Prevents floating-point errors
- Used in cryptocurrency and blockchain

**Intl API**:
- Language-sensitive formatting
- Number, date, and time localization
- Essential for international applications
- Supports multiple currencies and locales

---

**End of Questions 34-36**
