# Node.js Interview Question: ES Modules (ESM)
## Question 6: What are ES Modules? Understanding import/export and Modern JavaScript Modules

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **ES Modules vs CommonJS** - Key differences and when to use each
2. **import/export syntax** - Named exports, default exports, dynamic imports
3. **Static analysis & Tree-shaking** - How ESM enables better optimization
4. **Top-level await** - Using await outside async functions
5. **Interoperability** - Using ESM with CommonJS modules
6. **Banking Examples**: Modern payment API, transaction processor, account management

**Why This Matters in Banking**:
- Modern syntax for cleaner, more maintainable code
- Better performance through tree-shaking (smaller bundles)
- Static analysis catches errors at build time (not runtime)
- Top-level await simplifies database initialization
- Future-proof codebase (ESM is the JavaScript standard)

---

## 🎯 What are ES Modules?

ES Modules (ESM) are the **official JavaScript module standard** introduced in ES6 (ES2015).

### Key Characteristics:

```javascript
// ESM uses import and export
import { validate } from './validator.js';
export const process = () => { ... };

// Top-level await is supported
const config = await loadConfig();
```

**Core Concepts**:
1. **Static Structure**: Imports/exports are analyzed at parse time
2. **Asynchronous Loading**: Modules load asynchronously
3. **Strict Mode**: Always in strict mode (no `'use strict'` needed)
4. **Immutable Bindings**: Imported values are read-only references
5. **File Extension**: Must use `.mjs` or set `"type": "module"` in package.json

---

## 📊 ES Modules vs CommonJS - Complete Comparison

| Feature | CommonJS | ES Modules |
|---------|----------|------------|
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **Loading** | Synchronous | Asynchronous |
| **Analysis** | Runtime | Parse time (static) |
| **Tree-shaking** | ❌ Not possible | ✅ Fully supported |
| **Top-level await** | ❌ Not supported | ✅ Supported |
| **Circular deps** | ⚠️ Returns incomplete exports | ⚠️ Returns undefined |
| **File extension** | `.js` | `.mjs` or `"type": "module"` |
| **this** | `exports` object | `undefined` |
| **Dynamic imports** | ✅ Always dynamic | ✅ `import()` for dynamic |
| **Default in Node.js** | ✅ Yes (traditional) | ⚠️ Opt-in |
| **Browser support** | ❌ Needs bundler | ✅ Native support |

---

## 🔍 Deep Dive: import and export Syntax

### Named Exports

```javascript
// ✅ Export individual items
export const API_VERSION = '1.0.0';
export const MAX_RETRY = 3;

export function validateAmount(amount) {
  return amount > 0;
}

export class PaymentProcessor {
  process(payment) { ... }
}

// ✅ Export multiple at once
const API_VERSION = '1.0.0';
const MAX_RETRY = 3;
function validateAmount(amount) { ... }

export { API_VERSION, MAX_RETRY, validateAmount };

// ✅ Export with rename
export { validateAmount as validate };
```

### Default Exports

```javascript
// ✅ Export single default value
export default class PaymentService {
  process(payment) { ... }
}

// ✅ Export default with name
class PaymentService { ... }
export default PaymentService;

// ✅ Export inline
export default function processPayment(payment) { ... }

// ⚠️ Only ONE default export per module
```

### Import Syntax

```javascript
// ✅ Named imports
import { validate, format } from './utils.js';

// ✅ Import with rename
import { validate as validatePayment } from './validator.js';

// ✅ Import everything as namespace
import * as Utils from './utils.js';
Utils.validate();
Utils.format();

// ✅ Import default
import PaymentService from './payment-service.js';

// ✅ Mix default and named
import PaymentService, { validate, format } from './payment.js';

// ✅ Import for side effects only
import './polyfills.js';
```

### Dynamic Imports

```javascript
// ✅ Load module conditionally
if (needsPaymentProcessor) {
  const { PaymentProcessor } = await import('./payment-processor.js');
  const processor = new PaymentProcessor();
}

// ✅ Load module on demand
button.addEventListener('click', async () => {
  const { handlePayment } = await import('./payment-handler.js');
  await handlePayment();
});

// ✅ Conditional imports based on environment
const module = process.env.NODE_ENV === 'production'
  ? await import('./prod-config.js')
  : await import('./dev-config.js');
```

---

## 🏦 Example 1: Modern Banking Payment API with ES Modules

This example demonstrates ES Module syntax, static analysis benefits, and modern patterns.

**File: `package.json`**

```json
{
  "name": "banking-payment-api",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch src/server.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

**File: `src/constants/payment-types.js`**

```javascript
/**
 * Payment Type Constants
 * Demonstrates: Named exports with ES Modules
 */

// ✅ Export constants individually
export const PAYMENT_TYPES = {
  CREDIT_CARD: 'CREDIT_CARD',
  DEBIT_CARD: 'DEBIT_CARD',
  BANK_TRANSFER: 'BANK_TRANSFER',
  DIGITAL_WALLET: 'DIGITAL_WALLET',
  CRYPTOCURRENCY: 'CRYPTOCURRENCY'
};

export const PAYMENT_STATUS = {
  PENDING: 'PENDING',
  PROCESSING: 'PROCESSING',
  COMPLETED: 'COMPLETED',
  FAILED: 'FAILED',
  REFUNDED: 'REFUNDED'
};

export const CURRENCY_CODES = ['USD', 'EUR', 'GBP', 'JPY', 'INR'];

export const TRANSACTION_LIMITS = {
  MIN_AMOUNT: 0.01,
  MAX_AMOUNT: 1000000,
  DAILY_LIMIT: 50000
};

// ✅ Export utility functions
export function isValidPaymentType(type) {
  return Object.values(PAYMENT_TYPES).includes(type);
}

export function isValidStatus(status) {
  return Object.values(PAYMENT_STATUS).includes(status);
}

export function isValidCurrency(currency) {
  return CURRENCY_CODES.includes(currency);
}

console.log('✅ Payment constants module loaded');
```

**File: `src/validators/payment-validator.js`**

```javascript
/**
 * Payment Validator
 * Demonstrates: Importing from other ES Modules
 */

// ✅ Import specific named exports
import { 
  PAYMENT_TYPES, 
  TRANSACTION_LIMITS,
  isValidPaymentType,
  isValidCurrency 
} from '../constants/payment-types.js';

// ✅ Named export for validation function
export function validatePaymentAmount(amount) {
  const errors = [];
  
  if (typeof amount !== 'number' || isNaN(amount)) {
    errors.push('Amount must be a valid number');
  }
  
  if (amount < TRANSACTION_LIMITS.MIN_AMOUNT) {
    errors.push(`Amount must be at least ${TRANSACTION_LIMITS.MIN_AMOUNT}`);
  }
  
  if (amount > TRANSACTION_LIMITS.MAX_AMOUNT) {
    errors.push(`Amount cannot exceed ${TRANSACTION_LIMITS.MAX_AMOUNT}`);
  }
  
  return {
    valid: errors.length === 0,
    errors: errors.length > 0 ? errors : undefined
  };
}

export function validatePaymentMethod(method) {
  if (!isValidPaymentType(method)) {
    return {
      valid: false,
      error: `Invalid payment method. Must be one of: ${Object.values(PAYMENT_TYPES).join(', ')}`
    };
  }
  
  return { valid: true };
}

export function validatePaymentRequest(payment) {
  const errors = [];
  
  // Validate amount
  const amountResult = validatePaymentAmount(payment.amount);
  if (!amountResult.valid) {
    errors.push(...amountResult.errors);
  }
  
  // Validate method
  const methodResult = validatePaymentMethod(payment.method);
  if (!methodResult.valid) {
    errors.push(methodResult.error);
  }
  
  // Validate currency
  if (!isValidCurrency(payment.currency)) {
    errors.push(`Invalid currency code: ${payment.currency}`);
  }
  
  // Validate customer
  if (!payment.customerId) {
    errors.push('Customer ID is required');
  }
  
  return {
    valid: errors.length === 0,
    errors: errors.length > 0 ? errors : undefined
  };
}

console.log('✅ Payment validator module loaded');
```

**File: `src/services/payment-processor.js`**

```javascript
/**
 * Payment Processor Service
 * Demonstrates: Default export with class
 */

// ✅ Import from multiple modules
import { PAYMENT_STATUS, PAYMENT_TYPES } from '../constants/payment-types.js';
import { validatePaymentRequest } from '../validators/payment-validator.js';

// ✅ Default export of class
export default class PaymentProcessor {
  constructor(config = {}) {
    this.processingFee = config.processingFee || 0.029; // 2.9%
    this.fixedFee = config.fixedFee || 0.30;
    this.transactions = new Map();
    
    console.log('✅ PaymentProcessor initialized');
  }
  
  async processPayment(paymentRequest) {
    console.log(`\n💳 Processing payment: ${paymentRequest.method} - $${paymentRequest.amount}`);
    
    // Step 1: Validate payment request
    const validation = validatePaymentRequest(paymentRequest);
    if (!validation.valid) {
      console.error('❌ Validation failed:', validation.errors);
      return {
        success: false,
        status: PAYMENT_STATUS.FAILED,
        errors: validation.errors
      };
    }
    
    // Step 2: Calculate fees
    const fees = this.calculateFees(paymentRequest.amount);
    console.log(`   Fees: $${fees.toFixed(2)}`);
    
    // Step 3: Process based on payment method
    const result = await this.processPaymentMethod(
      paymentRequest.method,
      paymentRequest
    );
    
    // Step 4: Store transaction
    if (result.success) {
      this.transactions.set(result.transactionId, {
        ...paymentRequest,
        ...result,
        fees,
        timestamp: new Date()
      });
    }
    
    return result;
  }
  
  async processPaymentMethod(method, payment) {
    // Simulate payment gateway API call
    await new Promise(resolve => setTimeout(resolve, 100));
    
    const transactionId = `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    
    switch (method) {
      case PAYMENT_TYPES.CREDIT_CARD:
        return this.processCreditCard(payment, transactionId);
      
      case PAYMENT_TYPES.BANK_TRANSFER:
        return this.processBankTransfer(payment, transactionId);
      
      case PAYMENT_TYPES.DIGITAL_WALLET:
        return this.processDigitalWallet(payment, transactionId);
      
      default:
        return {
          success: false,
          status: PAYMENT_STATUS.FAILED,
          error: 'Unsupported payment method'
        };
    }
  }
  
  processCreditCard(payment, transactionId) {
    console.log('   Processing via credit card gateway...');
    
    return {
      success: true,
      transactionId,
      status: PAYMENT_STATUS.COMPLETED,
      method: PAYMENT_TYPES.CREDIT_CARD,
      amount: payment.amount,
      currency: payment.currency,
      last4: payment.cardDetails?.last4 || '****'
    };
  }
  
  processBankTransfer(payment, transactionId) {
    console.log('   Processing via bank transfer...');
    
    return {
      success: true,
      transactionId,
      status: PAYMENT_STATUS.PROCESSING, // Bank transfers take longer
      method: PAYMENT_TYPES.BANK_TRANSFER,
      amount: payment.amount,
      currency: payment.currency,
      estimatedCompletion: new Date(Date.now() + 24 * 60 * 60 * 1000) // 24 hours
    };
  }
  
  processDigitalWallet(payment, transactionId) {
    console.log('   Processing via digital wallet...');
    
    return {
      success: true,
      transactionId,
      status: PAYMENT_STATUS.COMPLETED,
      method: PAYMENT_TYPES.DIGITAL_WALLET,
      amount: payment.amount,
      currency: payment.currency,
      walletProvider: payment.walletProvider || 'UNKNOWN'
    };
  }
  
  calculateFees(amount) {
    return (amount * this.processingFee) + this.fixedFee;
  }
  
  getTransaction(transactionId) {
    return this.transactions.get(transactionId);
  }
  
  getAllTransactions() {
    return Array.from(this.transactions.values());
  }
  
  getStats() {
    const transactions = this.getAllTransactions();
    const successful = transactions.filter(t => t.success).length;
    const failed = transactions.filter(t => !t.success).length;
    const totalAmount = transactions
      .filter(t => t.success)
      .reduce((sum, t) => sum + t.amount, 0);
    const totalFees = transactions
      .filter(t => t.success)
      .reduce((sum, t) => sum + t.fees, 0);
    
    return {
      total: transactions.length,
      successful,
      failed,
      totalAmount: totalAmount.toFixed(2),
      totalFees: totalFees.toFixed(2),
      netAmount: (totalAmount - totalFees).toFixed(2)
    };
  }
}

console.log('✅ PaymentProcessor module loaded');
```

**File: `src/app.js`**

```javascript
/**
 * Main Application
 * Demonstrates: Top-level await and mixing default/named imports
 */

console.log('🚀 Starting Banking Payment API...\n');
console.log('='.repeat(70));

// ✅ Top-level await (no need for async wrapper!)
// This is a key feature of ES Modules

// Import default export
import PaymentProcessor from './services/payment-processor.js';

// Import named exports
import { PAYMENT_TYPES, CURRENCY_CODES } from './constants/payment-types.js';

// Initialize payment processor
const processor = new PaymentProcessor({
  processingFee: 0.029,
  fixedFee: 0.30
});

console.log('\n📊 Running Payment Tests:\n');
console.log('─'.repeat(70));

// Test 1: Credit Card Payment
console.log('\n🔸 Test 1: Credit Card Payment');
const payment1 = await processor.processPayment({
  customerId: 'CUST001',
  amount: 150.00,
  currency: 'USD',
  method: PAYMENT_TYPES.CREDIT_CARD,
  cardDetails: {
    last4: '4242'
  }
});
console.log('Result:', payment1);

// Test 2: Bank Transfer
console.log('\n🔸 Test 2: Bank Transfer');
const payment2 = await processor.processPayment({
  customerId: 'CUST002',
  amount: 5000.00,
  currency: 'USD',
  method: PAYMENT_TYPES.BANK_TRANSFER
});
console.log('Result:', payment2);

// Test 3: Digital Wallet
console.log('\n🔸 Test 3: Digital Wallet Payment');
const payment3 = await processor.processPayment({
  customerId: 'CUST003',
  amount: 75.50,
  currency: 'USD',
  method: PAYMENT_TYPES.DIGITAL_WALLET,
  walletProvider: 'PAYPAL'
});
console.log('Result:', payment3);

// Test 4: Invalid Payment (too small)
console.log('\n🔸 Test 4: Invalid Payment (amount too small)');
const payment4 = await processor.processPayment({
  customerId: 'CUST004',
  amount: 0.001, // Below minimum
  currency: 'USD',
  method: PAYMENT_TYPES.CREDIT_CARD
});
console.log('Result:', payment4);

// Test 5: Invalid Payment (missing customer)
console.log('\n🔸 Test 5: Invalid Payment (missing customer ID)');
const payment5 = await processor.processPayment({
  amount: 100.00,
  currency: 'USD',
  method: PAYMENT_TYPES.CREDIT_CARD
  // Missing customerId
});
console.log('Result:', payment5);

// Display statistics
console.log('\n' + '─'.repeat(70));
console.log('\n📈 Payment Statistics:\n');
const stats = processor.getStats();
console.log(`   Total Transactions: ${stats.total}`);
console.log(`   Successful: ${stats.successful}`);
console.log(`   Failed: ${stats.failed}`);
console.log(`   Total Amount: $${stats.totalAmount}`);
console.log(`   Total Fees: $${stats.totalFees}`);
console.log(`   Net Amount: $${stats.netAmount}`);

console.log('\n' + '='.repeat(70));
console.log('\n✅ Application completed successfully!\n');
```

**To test this example:**

```bash
# Ensure package.json has "type": "module"

# Run the application
node src/app.js

# Or with Node 18+ watch mode
node --watch src/app.js
```

**Expected Output:**

```
🚀 Starting Banking Payment API...

======================================================================
✅ Payment constants module loaded
✅ Payment validator module loaded
✅ PaymentProcessor module loaded
✅ PaymentProcessor initialized

📊 Running Payment Tests:

──────────────────────────────────────────────────────────────────────

🔸 Test 1: Credit Card Payment

💳 Processing payment: CREDIT_CARD - $150
   Fees: $4.65
   Processing via credit card gateway...
Result: {
  success: true,
  transactionId: 'TXN-1699876800123-abc123xyz',
  status: 'COMPLETED',
  method: 'CREDIT_CARD',
  amount: 150,
  currency: 'USD',
  last4: '4242'
}

🔸 Test 2: Bank Transfer

💳 Processing payment: BANK_TRANSFER - $5000
   Fees: $145.30
   Processing via bank transfer...
Result: {
  success: true,
  transactionId: 'TXN-1699876800234-def456uvw',
  status: 'PROCESSING',
  method: 'BANK_TRANSFER',
  amount: 5000,
  currency: 'USD',
  estimatedCompletion: 2025-11-14T10:40:00.000Z
}

🔸 Test 3: Digital Wallet Payment

💳 Processing payment: DIGITAL_WALLET - $75.5
   Fees: $2.49
   Processing via digital wallet...
Result: {
  success: true,
  transactionId: 'TXN-1699876800345-ghi789rst',
  status: 'COMPLETED',
  method: 'DIGITAL_WALLET',
  amount: 75.5,
  currency: 'USD',
  walletProvider: 'PAYPAL'
}

🔸 Test 4: Invalid Payment (amount too small)

💳 Processing payment: CREDIT_CARD - $0.001
❌ Validation failed: [ 'Amount must be at least 0.01' ]
Result: {
  success: false,
  status: 'FAILED',
  errors: [ 'Amount must be at least 0.01' ]
}

🔸 Test 5: Invalid Payment (missing customer ID)

💳 Processing payment: CREDIT_CARD - $100
❌ Validation failed: [ 'Customer ID is required' ]
Result: {
  success: false,
  status: 'FAILED',
  errors: [ 'Customer ID is required' ]
}

──────────────────────────────────────────────────────────────────────

📈 Payment Statistics:

   Total Transactions: 5
   Successful: 3
   Failed: 2
   Total Amount: $5225.50
   Total Fees: $152.44
   Net Amount: $5073.06

======================================================================

✅ Application completed successfully!
```

**Key Learnings from Example 1**:

1. **Top-level await**: No need for async wrapper in main file
2. **Named exports**: Multiple exports from one module
3. **Default export**: Single class export for PaymentProcessor
4. **Static imports**: Faster parsing, better optimization
5. **File extensions**: Must include `.js` in import paths
6. **Clean syntax**: Modern, readable code structure

---

## 🌳 Tree-Shaking and Dynamic Imports

### What is Tree-Shaking?

**Tree-shaking** is the process of removing unused code from your final bundle. ES Modules enable this because imports/exports are **static** and can be analyzed at build time.

```javascript
// utils.js - Export many functions
export function formatCurrency(amount) { ... }
export function formatDate(date) { ... }
export function formatName(name) { ... }
export function formatAddress(addr) { ... }  // Not used

// app.js - Only import what you need
import { formatCurrency, formatDate } from './utils.js';

// Result after tree-shaking:
// formatName and formatAddress are removed from bundle
// Smaller bundle size = faster load times
```

**Why CommonJS can't tree-shake:**

```javascript
// CommonJS is dynamic - can't analyze at build time
const utils = require('./utils');

// This could be anything:
const functionName = someCondition ? 'formatCurrency' : 'formatDate';
utils[functionName]();  // Impossible to know which function at build time

// So bundlers must include ALL exports
```

---

## 🏦 Example 2: Dynamic Imports and Code Splitting

This example shows how to use dynamic imports for on-demand loading and performance optimization.

**File: `src/processors/credit-card-processor.js`**

```javascript
/**
 * Credit Card Processor (Heavy module - 500KB)
 * Demonstrates: Module that should be loaded dynamically
 */

console.log('💳 Loading Credit Card Processor (simulating 500KB module)...');

// Simulate heavy dependencies
await new Promise(resolve => setTimeout(resolve, 100));

export default class CreditCardProcessor {
  constructor() {
    this.gatewayUrl = 'https://api.payment-gateway.com';
    console.log('✅ CreditCardProcessor initialized');
  }
  
  async validateCard(cardNumber, cvv, expiryDate) {
    console.log('   Validating card...');
    
    // Luhn algorithm for card validation
    const cleaned = cardNumber.replace(/\D/g, '');
    
    if (cleaned.length < 13 || cleaned.length > 19) {
      return { valid: false, error: 'Invalid card number length' };
    }
    
    // Check Luhn
    let sum = 0;
    let isEven = false;
    
    for (let i = cleaned.length - 1; i >= 0; i--) {
      let digit = parseInt(cleaned[i], 10);
      
      if (isEven) {
        digit *= 2;
        if (digit > 9) digit -= 9;
      }
      
      sum += digit;
      isEven = !isEven;
    }
    
    const luhnValid = sum % 10 === 0;
    
    if (!luhnValid) {
      return { valid: false, error: 'Invalid card number (checksum failed)' };
    }
    
    return { valid: true };
  }
  
  async processPayment(cardDetails, amount) {
    console.log(`   Processing credit card payment: $${amount}`);
    
    // Simulate API call to payment gateway
    await new Promise(resolve => setTimeout(resolve, 200));
    
    return {
      success: true,
      transactionId: `CC-${Date.now()}`,
      amount,
      last4: cardDetails.number.slice(-4),
      timestamp: new Date()
    };
  }
  
  getProviderInfo() {
    return {
      name: 'Credit Card Gateway',
      supportedBrands: ['VISA', 'MASTERCARD', 'AMEX', 'DISCOVER'],
      maxAmount: 50000,
      processingTime: '2-5 seconds'
    };
  }
}

console.log('✅ CreditCardProcessor module fully loaded');
```

**File: `src/processors/crypto-processor.js`**

```javascript
/**
 * Cryptocurrency Processor (Very heavy - 2MB)
 * Demonstrates: Module that should definitely be loaded on-demand
 */

console.log('₿ Loading Cryptocurrency Processor (simulating 2MB module)...');

// Simulate very heavy dependencies (blockchain libraries, etc.)
await new Promise(resolve => setTimeout(resolve, 300));

export default class CryptoProcessor {
  constructor() {
    this.supportedCoins = ['BTC', 'ETH', 'USDT', 'BNB'];
    console.log('✅ CryptoProcessor initialized');
  }
  
  async getExchangeRate(crypto, fiat = 'USD') {
    console.log(`   Fetching ${crypto}/${fiat} exchange rate...`);
    
    // Simulate API call to crypto exchange
    await new Promise(resolve => setTimeout(resolve, 150));
    
    // Mock rates
    const rates = {
      'BTC/USD': 45000,
      'ETH/USD': 3200,
      'USDT/USD': 1,
      'BNB/USD': 580
    };
    
    return rates[`${crypto}/${fiat}`] || 0;
  }
  
  async processPayment(cryptoDetails, amountInFiat) {
    console.log(`   Processing crypto payment: $${amountInFiat}`);
    
    const { crypto, walletAddress } = cryptoDetails;
    
    // Get current exchange rate
    const rate = await this.getExchangeRate(crypto, 'USD');
    const cryptoAmount = amountInFiat / rate;
    
    // Simulate blockchain transaction
    await new Promise(resolve => setTimeout(resolve, 300));
    
    return {
      success: true,
      transactionId: `CRYPTO-${Date.now()}`,
      fiatAmount: amountInFiat,
      cryptoAmount: cryptoAmount.toFixed(8),
      crypto,
      walletAddress,
      txHash: `0x${Math.random().toString(16).substr(2, 64)}`,
      timestamp: new Date()
    };
  }
  
  getProviderInfo() {
    return {
      name: 'Cryptocurrency Gateway',
      supportedCoins: this.supportedCoins,
      maxAmount: 1000000,
      processingTime: '10-60 minutes (blockchain confirmations)'
    };
  }
}

console.log('✅ CryptoProcessor module fully loaded');
```

**File: `src/payment-router.js`**

```javascript
/**
 * Payment Router with Dynamic Imports
 * Demonstrates: Loading heavy modules only when needed
 */

import { PAYMENT_TYPES } from './constants/payment-types.js';

export default class PaymentRouter {
  constructor() {
    this.processedCount = 0;
    this.loadedProcessors = new Set();
    console.log('✅ PaymentRouter initialized');
  }
  
  async routePayment(paymentRequest) {
    console.log(`\n🔀 Routing payment: ${paymentRequest.method}`);
    
    this.processedCount++;
    
    switch (paymentRequest.method) {
      case PAYMENT_TYPES.CREDIT_CARD:
        return await this.processCreditCard(paymentRequest);
      
      case PAYMENT_TYPES.CRYPTOCURRENCY:
        return await this.processCrypto(paymentRequest);
      
      case PAYMENT_TYPES.BANK_TRANSFER:
        return await this.processBankTransfer(paymentRequest);
      
      default:
        throw new Error(`Unsupported payment method: ${paymentRequest.method}`);
    }
  }
  
  async processCreditCard(payment) {
    console.log('   Loading Credit Card Processor dynamically...');
    
    // ✅ Dynamic import - only load when needed!
    const { default: CreditCardProcessor } = await import('./processors/credit-card-processor.js');
    
    this.loadedProcessors.add('CreditCardProcessor');
    
    const processor = new CreditCardProcessor();
    
    // Validate card
    const validation = await processor.validateCard(
      payment.cardDetails.number,
      payment.cardDetails.cvv,
      payment.cardDetails.expiry
    );
    
    if (!validation.valid) {
      return {
        success: false,
        error: validation.error
      };
    }
    
    // Process payment
    return await processor.processPayment(payment.cardDetails, payment.amount);
  }
  
  async processCrypto(payment) {
    console.log('   Loading Cryptocurrency Processor dynamically...');
    
    // ✅ Dynamic import - only load when needed!
    // This is a 2MB module - we don't want to load it unless necessary
    const { default: CryptoProcessor } = await import('./processors/crypto-processor.js');
    
    this.loadedProcessors.add('CryptoProcessor');
    
    const processor = new CryptoProcessor();
    
    return await processor.processPayment(payment.cryptoDetails, payment.amount);
  }
  
  async processBankTransfer(payment) {
    console.log('   Processing bank transfer (lightweight)...');
    
    // Simple inline processing - no need for separate module
    await new Promise(resolve => setTimeout(resolve, 100));
    
    return {
      success: true,
      transactionId: `BT-${Date.now()}`,
      amount: payment.amount,
      method: PAYMENT_TYPES.BANK_TRANSFER,
      estimatedCompletion: new Date(Date.now() + 24 * 60 * 60 * 1000)
    };
  }
  
  getStats() {
    return {
      processedCount: this.processedCount,
      loadedProcessors: Array.from(this.loadedProcessors),
      memoryUsage: process.memoryUsage().heapUsed / 1024 / 1024
    };
  }
}

console.log('✅ PaymentRouter module loaded');
```

**File: `src/test-dynamic-imports.js`**

```javascript
/**
 * Test Dynamic Imports
 * Demonstrates: Performance benefits of dynamic imports
 */

console.log('🧪 Testing Dynamic Imports vs Static Imports\n');
console.log('='.repeat(70));

import PaymentRouter from './payment-router.js';
import { PAYMENT_TYPES } from './constants/payment-types.js';

const router = new PaymentRouter();

console.log('\n📊 Initial State:');
console.log('   Only PaymentRouter is loaded');
console.log('   Heavy processors (CreditCard, Crypto) are NOT loaded yet');
console.log(`   Memory usage: ${(process.memoryUsage().heapUsed / 1024 / 1024).toFixed(2)} MB\n`);

console.log('─'.repeat(70));

// Test 1: Bank Transfer (lightweight, no dynamic import)
console.log('\n🔸 Test 1: Bank Transfer (No Dynamic Import Needed)');
const payment1 = await router.routePayment({
  method: PAYMENT_TYPES.BANK_TRANSFER,
  amount: 1000,
  accountNumber: '1234567890'
});
console.log('Result:', payment1);

let stats = router.getStats();
console.log(`\nStats after Test 1:`);
console.log(`   Processed: ${stats.processedCount}`);
console.log(`   Loaded processors: ${stats.loadedProcessors.join(', ') || 'None'}`);
console.log(`   Memory usage: ${stats.memoryUsage.toFixed(2)} MB`);

console.log('\n─'.repeat(70));

// Test 2: Credit Card (triggers dynamic import)
console.log('\n🔸 Test 2: Credit Card Payment (Triggers Dynamic Import)');
console.log('Watch: CreditCardProcessor will be loaded now...\n');

const payment2 = await router.routePayment({
  method: PAYMENT_TYPES.CREDIT_CARD,
  amount: 250,
  cardDetails: {
    number: '4532015112830366',  // Valid test card
    cvv: '123',
    expiry: '12/25'
  }
});
console.log('\nResult:', payment2);

stats = router.getStats();
console.log(`\nStats after Test 2:`);
console.log(`   Processed: ${stats.processedCount}`);
console.log(`   Loaded processors: ${stats.loadedProcessors.join(', ')}`);
console.log(`   Memory usage: ${stats.memoryUsage.toFixed(2)} MB`);

console.log('\n─'.repeat(70));

// Test 3: Cryptocurrency (triggers another dynamic import)
console.log('\n🔸 Test 3: Cryptocurrency Payment (Triggers Another Dynamic Import)');
console.log('Watch: CryptoProcessor (2MB) will be loaded now...\n');

const payment3 = await router.routePayment({
  method: PAYMENT_TYPES.CRYPTOCURRENCY,
  amount: 500,
  cryptoDetails: {
    crypto: 'BTC',
    walletAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'
  }
});
console.log('\nResult:', payment3);

stats = router.getStats();
console.log(`\nStats after Test 3:`);
console.log(`   Processed: ${stats.processedCount}`);
console.log(`   Loaded processors: ${stats.loadedProcessors.join(', ')}`);
console.log(`   Memory usage: ${stats.memoryUsage.toFixed(2)} MB`);

console.log('\n─'.repeat(70));

// Test 4: Another credit card payment (processor already loaded)
console.log('\n🔸 Test 4: Another Credit Card Payment (Already Loaded)');
console.log('Watch: CreditCardProcessor is reused, not loaded again...\n');

const payment4 = await router.routePayment({
  method: PAYMENT_TYPES.CREDIT_CARD,
  amount: 100,
  cardDetails: {
    number: '5425233430109903',  // Valid test card
    cvv: '456',
    expiry: '06/26'
  }
});
console.log('\nResult:', payment4);

stats = router.getStats();
console.log(`\nStats after Test 4:`);
console.log(`   Processed: ${stats.processedCount}`);
console.log(`   Loaded processors: ${stats.loadedProcessors.join(', ')}`);
console.log(`   Memory usage: ${stats.memoryUsage.toFixed(2)} MB`);

console.log('\n' + '='.repeat(70));
console.log('\n💡 Key Insights:\n');
console.log('   ✅ Processors loaded only when needed');
console.log('   ✅ Faster initial load time');
console.log('   ✅ Lower memory usage (if processor never used)');
console.log('   ✅ Once loaded, cached and reused');
console.log('   ✅ Perfect for supporting many payment methods');
console.log('\n' + '='.repeat(70) + '\n');
```

**To test this example:**

```bash
node src/test-dynamic-imports.js
```

**Expected Output:**

```
🧪 Testing Dynamic Imports vs Static Imports

======================================================================
✅ PaymentRouter module loaded
✅ PaymentRouter initialized

📊 Initial State:
   Only PaymentRouter is loaded
   Heavy processors (CreditCard, Crypto) are NOT loaded yet
   Memory usage: 15.23 MB

──────────────────────────────────────────────────────────────────────

🔸 Test 1: Bank Transfer (No Dynamic Import Needed)

🔀 Routing payment: BANK_TRANSFER
   Processing bank transfer (lightweight)...
Result: {
  success: true,
  transactionId: 'BT-1699877400123',
  amount: 1000,
  method: 'BANK_TRANSFER',
  estimatedCompletion: 2025-11-14T10:45:00.000Z
}

Stats after Test 1:
   Processed: 1
   Loaded processors: None
   Memory usage: 15.45 MB

──────────────────────────────────────────────────────────────────────

🔸 Test 2: Credit Card Payment (Triggers Dynamic Import)
Watch: CreditCardProcessor will be loaded now...

🔀 Routing payment: CREDIT_CARD
   Loading Credit Card Processor dynamically...
💳 Loading Credit Card Processor (simulating 500KB module)...
✅ CreditCardProcessor initialized
✅ CreditCardProcessor module fully loaded
   Validating card...
   Processing credit card payment: $250

Result: {
  success: true,
  transactionId: 'CC-1699877400234',
  amount: 250,
  last4: '0366',
  timestamp: 2025-11-13T10:45:00.234Z
}

Stats after Test 2:
   Processed: 2
   Loaded processors: CreditCardProcessor
   Memory usage: 16.12 MB

──────────────────────────────────────────────────────────────────────

🔸 Test 3: Cryptocurrency Payment (Triggers Another Dynamic Import)
Watch: CryptoProcessor (2MB) will be loaded now...

🔀 Routing payment: CRYPTOCURRENCY
   Loading Cryptocurrency Processor dynamically...
₿ Loading Cryptocurrency Processor (simulating 2MB module)...
✅ CryptoProcessor initialized
✅ CryptoProcessor module fully loaded
   Processing crypto payment: $500
   Fetching BTC/USD exchange rate...

Result: {
  success: true,
  transactionId: 'CRYPTO-1699877400567',
  fiatAmount: 500,
  cryptoAmount: '0.01111111',
  crypto: 'BTC',
  walletAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
  txHash: '0x7f9fade1c0d57a7af66ab4ead79fade1c0d57a7af66ab4ead7c2c2eb7b11a91385',
  timestamp: 2025-11-13T10:45:00.567Z
}

Stats after Test 3:
   Processed: 3
   Loaded processors: CreditCardProcessor, CryptoProcessor
   Memory usage: 18.89 MB

──────────────────────────────────────────────────────────────────────

🔸 Test 4: Another Credit Card Payment (Already Loaded)
Watch: CreditCardProcessor is reused, not loaded again...

🔀 Routing payment: CREDIT_CARD
   Loading Credit Card Processor dynamically...
✅ CreditCardProcessor initialized
   Validating card...
   Processing credit card payment: $100

Result: {
  success: true,
  transactionId: 'CC-1699877400789',
  amount: 100,
  last4: '9903',
  timestamp: 2025-11-13T10:45:00.789Z
}

Stats after Test 4:
   Processed: 4
   Loaded processors: CreditCardProcessor, CryptoProcessor
   Memory usage: 18.95 MB

======================================================================

💡 Key Insights:

   ✅ Processors loaded only when needed
   ✅ Faster initial load time
   ✅ Lower memory usage (if processor never used)
   ✅ Once loaded, cached and reused
   ✅ Perfect for supporting many payment methods

======================================================================
```

**Key Learnings from Example 2**:

1. **Dynamic Imports**: Use `await import()` for on-demand loading
2. **Code Splitting**: Heavy modules loaded only when needed
3. **Performance**: Faster initial load, lower memory if feature unused
4. **Caching**: Once imported, module is cached
5. **Use Cases**: Payment processors, admin features, reports, heavy libraries

---

## 🔄 Example 3: ESM and CommonJS Interoperability

Node.js allows mixing ESM and CommonJS, but with some caveats.

### Importing CommonJS from ESM

**File: `legacy/account-service.js` (CommonJS)**

```javascript
// CommonJS module (old codebase)
class AccountService {
  getAccount(id) {
    return {
      id,
      balance: 1000,
      type: 'SAVINGS'
    };
  }
  
  updateBalance(id, amount) {
    console.log(`Updated account ${id} by $${amount}`);
    return true;
  }
}

module.exports = new AccountService();
```

**File: `modern/transaction-service.js` (ESM)**

```javascript
// ✅ ESM can import CommonJS
import accountService from '../legacy/account-service.js';

export async function processTransaction(accountId, amount) {
  const account = accountService.getAccount(accountId);
  accountService.updateBalance(accountId, amount);
  
  return {
    success: true,
    accountId,
    previousBalance: account.balance,
    newBalance: account.balance + amount
  };
}

// Named exports work fine
export const TRANSACTION_TYPES = ['DEBIT', 'CREDIT'];
```

### ❌ CommonJS CANNOT import ESM (synchronous issue)

```javascript
// ❌ This doesn't work in CommonJS
const esmModule = require('./esm-module.mjs');  // Error!

// ✅ Use dynamic import instead
(async () => {
  const esmModule = await import('./esm-module.mjs');
  esmModule.doSomething();
})();
```

---

## 10 DO's - Best Practices for ES Modules

### 1. ✅ DO: Always Include File Extensions

```javascript
// ✅ GOOD
import { validate } from './validator.js';
import PaymentService from './services/payment.js';

// ❌ BAD - May not work in Node.js ESM
import { validate } from './validator';
```

### 2. ✅ DO: Use Named Exports for Multiple Items

```javascript
// ✅ GOOD: Easy to tree-shake
export const API_KEY = 'abc123';
export const API_URL = 'https://api.example.com';
export function authenticate() { ... }

// Import only what you need
import { API_KEY } from './config.js';  // Tree-shaking works
```

### 3. ✅ DO: Use Default Export for Single Main Export

```javascript
// ✅ GOOD: One main class
export default class PaymentService {
  process() { ... }
}

// ✅ GOOD: One main function
export default function processPayment() { ... }
```

### 4. ✅ DO: Use Top-Level Await for Async Initialization

```javascript
// ✅ GOOD: Clean initialization
import { createConnection } from './database.js';

const db = await createConnection();

export async function query(sql) {
  return await db.execute(sql);
}

// No need for IIFE or async wrapper!
```

### 5. ✅ DO: Use Dynamic Imports for Code Splitting

```javascript
// ✅ GOOD: Load heavy modules on demand
button.addEventListener('click', async () => {
  const { generateReport } = await import('./report-generator.js');
  await generateReport();
});

// ✅ GOOD: Conditional loading
if (user.isAdmin) {
  const { AdminPanel } = await import('./admin.js');
  new AdminPanel();
}
```

### 6. ✅ DO: Organize with Barrel Exports (index.js)

```javascript
// validators/index.js
export { validateAccount } from './account-validator.js';
export { validateTransaction } from './transaction-validator.js';
export { validatePayment } from './payment-validator.js';

// Clean imports
import { validateAccount, validateTransaction } from './validators/index.js';
```

### 7. ✅ DO: Use "type": "module" in package.json

```json
{
  "type": "module",
  "main": "index.js"
}
```

### 8. ✅ DO: Handle Import Errors Gracefully

```javascript
// ✅ GOOD: Try-catch for dynamic imports
try {
  const module = await import('./optional-feature.js');
  module.initialize();
} catch (err) {
  console.warn('Optional feature unavailable:', err.message);
}
```

### 9. ✅ DO: Export Constants as Named Exports

```javascript
// ✅ GOOD: Tree-shakeable constants
export const STATUS_PENDING = 'PENDING';
export const STATUS_COMPLETED = 'COMPLETED';
export const STATUS_FAILED = 'FAILED';
```

### 10. ✅ DO: Use ESM for New Projects

```javascript
// ✅ GOOD: Start new projects with ESM
// - Better tooling support
// - Future-proof
// - Tree-shaking enabled
// - Top-level await
// - Static analysis
```

---

## 10 DON'Ts - Common Mistakes to Avoid

### 1. ❌ DON'T: Forget File Extensions

```javascript
// ❌ BAD: May not work
import { validate } from './validator';

// ✅ GOOD: Always include .js
import { validate } from './validator.js';
```

### 2. ❌ DON'T: Mix Default and Named Exports Unnecessarily

```javascript
// ❌ BAD: Confusing
export default class PaymentService { ... }
export const helper = () => { ... };

// ✅ GOOD: Pick one pattern
export class PaymentService { ... }
export const helper = () => { ... };
```

### 3. ❌ DON'T: Use require() in ESM

```javascript
// ❌ BAD: require is not defined in ESM
const fs = require('fs');

// ✅ GOOD: Use import
import fs from 'fs';
```

### 4. ❌ DON'T: Use __dirname or __filename Directly

```javascript
// ❌ BAD: Not available in ESM
const filePath = __dirname + '/file.txt';

// ✅ GOOD: Use import.meta.url
import { fileURLToPath } from 'url';
import { dirname } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

### 5. ❌ DON'T: Expect Synchronous require() Behavior

```javascript
// ❌ BAD: Trying to use require() pattern
const config = import('./config.js');  // Returns Promise!

// ✅ GOOD: Use await
const config = await import('./config.js');
```

### 6. ❌ DON'T: Import CommonJS Named Exports Directly

```javascript
// CommonJS: module.exports = { a: 1, b: 2 };

// ❌ BAD: May not work
import { a, b } from './commonjs-module.js';

// ✅ GOOD: Import default, then destructure
import module from './commonjs-module.js';
const { a, b } = module;
```

### 7. ❌ DON'T: Use Circular Dependencies

```javascript
// ❌ BAD: Circular dependencies
// a.js
import { b } from './b.js';
export const a = b + 1;

// b.js
import { a } from './a.js';  // a is undefined!
export const b = a + 1;

// ✅ GOOD: Restructure to avoid cycles
```

### 8. ❌ DON'T: Mutate Imported Bindings

```javascript
// ❌ BAD: Imported bindings are read-only
import { config } from './config.js';
config.apiKey = 'new-key';  // TypeError!

// ✅ GOOD: Create a copy
import { config as originalConfig } from './config.js';
const config = { ...originalConfig };
config.apiKey = 'new-key';
```

### 9. ❌ DON'T: Forget Top-Level Await is Blocking

```javascript
// ❌ BAD: Blocking main thread
const data = await fetch('https://slow-api.com');  // Blocks everything

// ✅ GOOD: Load in background or defer
setTimeout(async () => {
  const data = await fetch('https://slow-api.com');
}, 0);
```

### 10. ❌ DON'T: Mix .js and .mjs Without Understanding

```javascript
// ❌ BAD: Inconsistent extensions
import a from './module.js';   // ESM
import b from './other.mjs';   // Also ESM
import c from './legacy.cjs';  // CommonJS

// ✅ GOOD: Use "type": "module" and consistent .js
// Or use .mjs for ESM, .cjs for CommonJS
```

---

## 🎯 Key Takeaways

### ESM vs CommonJS Quick Reference

| Aspect | CommonJS | ES Modules |
|--------|----------|------------|
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **When Analyzed** | Runtime | Parse time |
| **Loading** | Synchronous | Asynchronous |
| **Top-level await** | ❌ No | ✅ Yes |
| **Tree-shaking** | ❌ No | ✅ Yes |
| **this** | exports object | undefined |
| **Dynamic** | Always | `import()` function |
| **File ext** | `.js` | `.js` with `"type": "module"` or `.mjs` |
| **Browser** | ❌ Needs bundler | ✅ Native |
| **__dirname** | ✅ Available | ❌ Must use `import.meta.url` |
| **Circular deps** | Incomplete exports | undefined |
| **Future** | Legacy | ✅ Standard |

### When to Use Each

| Use CommonJS When | Use ES Modules When |
|-------------------|---------------------|
| Legacy codebase | New projects |
| Node.js only | Browser + Node.js |
| Simple scripts | Complex applications |
| No build step | Using bundlers |
| Third-party requires it | Want tree-shaking |

### Import Patterns Comparison

```javascript
// CommonJS
const validator = require('./validator');
const { validate } = require('./validator');
const Processor = require('./processor');

// ES Modules
import validator from './validator.js';
import { validate } from './validator.js';
import Processor from './processor.js';

// Dynamic
const validator = require(condition ? './a' : './b');
const validator = await import(condition ? './a.js' : './b.js');
```

### Performance Impact

| Feature | Impact | Example |
|---------|--------|---------|
| **Static imports** | Faster parsing | `import { x } from './y.js'` |
| **Tree-shaking** | Smaller bundles | Removes unused exports |
| **Dynamic imports** | Code splitting | Loads on demand |
| **Top-level await** | Simplified async | No IIFE needed |

### Common Interview Questions

**Q: Can ESM import CommonJS?**  
A: Yes, ESM can import CommonJS. Default import gets the entire module.exports.

**Q: Can CommonJS import ESM?**  
A: No, not with require(). Must use dynamic import: `await import()`.

**Q: What is tree-shaking?**  
A: Removing unused code at build time. Only possible with ESM due to static structure.

**Q: Do I need file extensions in ESM?**  
A: Yes, must include `.js` in Node.js ESM (browsers too).

**Q: What is top-level await?**  
A: Using `await` outside async functions. Only available in ESM.

**Q: How to get __dirname in ESM?**  
A: Use `import.meta.url` with `fileURLToPath()` and `dirname()`.

**Q: Should I use .mjs or .js?**  
A: Use `.js` with `"type": "module"` in package.json (cleaner).

**Q: Can I mix ESM and CommonJS in one project?**  
A: Yes, but ESM must use dynamic import for CommonJS. Use `.mjs` and `.cjs` extensions if needed.

---

## 📊 Real-World Banking Application Structure

```
banking-api/
├── package.json              ("type": "module")
├── src/
│   ├── server.js            (Top-level await for DB connection)
│   │
│   ├── constants/
│   │   ├── index.js         (Barrel export)
│   │   ├── payment-types.js (Named exports)
│   │   ├── currencies.js    (Named exports)
│   │   └── limits.js        (Named exports)
│   │
│   ├── validators/
│   │   ├── index.js         (Barrel export)
│   │   ├── account.js       (Named exports)
│   │   ├── transaction.js   (Named exports)
│   │   └── payment.js       (Named exports)
│   │
│   ├── services/
│   │   ├── account.js       (Default export class)
│   │   ├── transaction.js   (Default export class)
│   │   └── payment.js       (Default export class)
│   │
│   ├── processors/
│   │   ├── credit-card.js   (Dynamic import - heavy)
│   │   ├── crypto.js        (Dynamic import - very heavy)
│   │   └── bank-transfer.js (Static import - lightweight)
│   │
│   ├── utils/
│   │   ├── index.js         (Barrel export)
│   │   ├── formatters.js    (Named exports)
│   │   └── calculators.js   (Named exports)
│   │
│   └── legacy/              (CommonJS modules)
│       ├── old-validator.js (module.exports)
│       └── old-service.js   (module.exports)
│
└── tests/
    └── *.test.js            (ESM with top-level await)
```

---

## 🏁 Conclusion

ES Modules are the **future of JavaScript** and offer significant advantages over CommonJS:

**Key Benefits**:
1. ✅ **Static Analysis**: Errors caught at parse time, not runtime
2. ✅ **Tree-Shaking**: Smaller bundles, faster load times
3. ✅ **Top-Level Await**: Cleaner async initialization
4. ✅ **Browser Native**: No bundler needed for modern browsers
5. ✅ **Better Tooling**: IDEs provide better autocomplete and refactoring
6. ✅ **Standard**: Official JavaScript spec, future-proof

**For Banking Applications**:
- Use ESM for new microservices
- Static imports for core modules (fast parsing)
- Dynamic imports for heavy processors (code splitting)
- Top-level await for database connections
- Tree-shaking for smaller deployment bundles
- Better security through static analysis

**Migration Path**:
1. Start new projects with ESM
2. Gradually migrate CommonJS modules
3. Use `.mjs` extension during transition
4. Keep CommonJS for Node.js-only scripts (if needed)
5. Test thoroughly (circular dependencies behave differently)

**Remember**:
- Always include `.js` extensions
- Use `"type": "module"` in package.json
- ESM can import CommonJS, but not vice versa (without dynamic import)
- Top-level await simplifies async code
- Tree-shaking reduces bundle size significantly

---

**End of Question 6: ES Modules (ESM)**

---



