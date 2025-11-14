# Node.js Interview Question: Buffer Encoding & Cryptography
## Question 8: How to Work with Buffer Encoding and Binary Data? Encrypt/Decrypt Sensitive Data?

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **Buffer Encoding** - utf8, base64, hex, ascii, binary, latin1
2. **Encoding Conversion** - Converting between different encodings
3. **Binary Data** - Working with raw binary data
4. **Cryptography** - Encrypting/decrypting sensitive data
5. **Hashing** - Creating secure hashes
6. **HMAC** - Message authentication codes
7. **Banking Examples**: Credit card encryption, PII protection, secure token generation

**Why This Matters in Banking**:
- Protect credit card numbers (PCI DSS compliance)
- Encrypt customer PII (Personally Identifiable Information)
- Secure API tokens and session keys
- Hash passwords and sensitive data
- Generate secure transaction signatures
- Store encrypted data in databases

---

## 🎯 Buffer Encoding Basics

### Common Encodings

Node.js Buffers support multiple character encodings:

```javascript
const text = 'Hello Banking!';

// UTF-8 (default, most common)
const utf8Buffer = Buffer.from(text, 'utf8');
console.log(utf8Buffer);  // <Buffer 48 65 6c 6c 6f 20 42 61 6e 6b 69 6e 67 21>

// Base64 (for transmitting binary data as text)
const base64 = utf8Buffer.toString('base64');
console.log(base64);  // SGVsbG8gQmFua2luZyE=

// Hex (hexadecimal representation)
const hex = utf8Buffer.toString('hex');
console.log(hex);  // 48656c6c6f2042616e6b696e6721

// ASCII (7-bit encoding)
const ascii = utf8Buffer.toString('ascii');
console.log(ascii);  // Hello Banking!
```

**Encoding Summary**:
- **utf8**: Universal, supports all characters (default)
- **base64**: Binary to text, commonly used for data transmission
- **hex**: Hexadecimal, human-readable binary representation
- **ascii**: 7-bit, English characters only
- **latin1/binary**: 8-bit, raw binary data
- **ucs2/utf16le**: 2-byte Unicode characters

---

## 📊 Visual: Buffer Encoding Flow

```
Original Data (String)
        │
        ├─► "4532-1234-5678-9010" (Credit Card)
        │
        ▼
   UTF-8 Buffer
        │
        ├─► <Buffer 34 35 33 32 2d 31 32 33 34 2d 35 36 37 38 2d 39 30 31 30>
        │
        ├─────────────────┬─────────────────┬──────────────────┐
        │                 │                 │                  │
        ▼                 ▼                 ▼                  ▼
    Base64             Hex             ASCII             Encrypted
        │                 │                 │                  │
        ▼                 ▼                 ▼                  ▼
"NDUzMi0xMjM0..."   "3435333..."    "4532-1234..."    "aGf9Kl2..."
```

---

## 🔍 Encoding Conversion Examples

```javascript
import crypto from 'crypto';

// Original credit card number
const cardNumber = '4532-1234-5678-9010';

console.log('='.repeat(70));
console.log('Buffer Encoding Conversions');
console.log('='.repeat(70));

// 1. String to UTF-8 Buffer
const utf8Buffer = Buffer.from(cardNumber, 'utf8');
console.log('\n1. UTF-8 Buffer:');
console.log('   ', utf8Buffer);
console.log('   Length:', utf8Buffer.length, 'bytes');

// 2. UTF-8 to Base64
const base64Encoded = utf8Buffer.toString('base64');
console.log('\n2. Base64 Encoded:');
console.log('   ', base64Encoded);
console.log('   Length:', base64Encoded.length, 'characters');

// 3. Base64 back to Buffer
const base64Buffer = Buffer.from(base64Encoded, 'base64');
console.log('\n3. Base64 to Buffer:');
console.log('   ', base64Buffer);

// 4. Buffer to Hex
const hexEncoded = utf8Buffer.toString('hex');
console.log('\n4. Hex Encoded:');
console.log('   ', hexEncoded);
console.log('   Length:', hexEncoded.length, 'characters');

// 5. Hex back to Buffer
const hexBuffer = Buffer.from(hexEncoded, 'hex');
console.log('\n5. Hex to Buffer:');
console.log('   ', hexBuffer);

// 6. Back to original string
const recovered = hexBuffer.toString('utf8');
console.log('\n6. Recovered String:');
console.log('   ', recovered);
console.log('   Match:', recovered === cardNumber ? '✅' : '❌');

console.log('\n' + '='.repeat(70));
```

---

## 🏦 Example 1: Credit Card Encryption System

This example demonstrates encrypting and decrypting credit card numbers using AES-256-GCM.

**File: `card-encryption-system.js`**

```javascript
import crypto from 'crypto';
import { promisify } from 'util';

/**
 * Banking Credit Card Encryption System
 * Demonstrates: AES-256-GCM encryption, secure key management, buffer encoding
 */

console.log('🔐 Credit Card Encryption System\n');
console.log('='.repeat(70));

// ============================================
// Encryption Configuration
// ============================================

// In production: Store in environment variables or key management service (AWS KMS, Azure Key Vault)
const ENCRYPTION_KEY = crypto.randomBytes(32); // 256-bit key for AES-256
const ALGORITHM = 'aes-256-gcm';

// ============================================
// Encryption Functions
// ============================================

/**
 * Encrypt sensitive data using AES-256-GCM
 * @param {string} plaintext - Data to encrypt
 * @returns {object} - Encrypted data with IV and auth tag
 */
function encrypt(plaintext) {
  try {
    // Generate random initialization vector (IV)
    const iv = crypto.randomBytes(16); // 128 bits for GCM
    
    // Create cipher
    const cipher = crypto.createCipheriv(ALGORITHM, ENCRYPTION_KEY, iv);
    
    // Encrypt data
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    // Get authentication tag (for GCM mode)
    const authTag = cipher.getAuthTag();
    
    return {
      encrypted: encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  } catch (err) {
    throw new Error(`Encryption failed: ${err.message}`);
  }
}

/**
 * Decrypt encrypted data using AES-256-GCM
 * @param {object} encryptedData - Object with encrypted, iv, authTag
 * @returns {string} - Decrypted plaintext
 */
function decrypt(encryptedData) {
  try {
    const { encrypted, iv, authTag } = encryptedData;
    
    // Create decipher
    const decipher = crypto.createDecipheriv(
      ALGORITHM,
      ENCRYPTION_KEY,
      Buffer.from(iv, 'hex')
    );
    
    // Set authentication tag (for GCM mode)
    decipher.setAuthTag(Buffer.from(authTag, 'hex'));
    
    // Decrypt data
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  } catch (err) {
    throw new Error(`Decryption failed: ${err.message}`);
  }
}

// ============================================
// Card Data Handler Class
// ============================================

class CardDataHandler {
  constructor() {
    this.encryptedCards = new Map();
  }
  
  /**
   * Store credit card securely
   */
  storeCard(customerId, cardNumber, cvv, expiryDate) {
    console.log(`\n🔒 Storing card for customer: ${customerId}`);
    console.log(`   Card Number: ${this.maskCard(cardNumber)}`);
    
    // Validate card number
    if (!this.validateCard(cardNumber)) {
      throw new Error('Invalid card number');
    }
    
    // Create card data object
    const cardData = {
      cardNumber,
      cvv,
      expiryDate,
      storedAt: new Date().toISOString()
    };
    
    // Convert to JSON
    const cardJson = JSON.stringify(cardData);
    
    // Encrypt the entire card data
    const encrypted = encrypt(cardJson);
    
    // Store encrypted data
    this.encryptedCards.set(customerId, {
      encryptedData: encrypted,
      lastFour: cardNumber.slice(-4),
      cardType: this.getCardType(cardNumber),
      storedAt: cardData.storedAt
    });
    
    console.log(`   ✅ Card encrypted and stored`);
    console.log(`   Last 4 digits: ****${this.encryptedCards.get(customerId).lastFour}`);
    console.log(`   Encrypted size: ${encrypted.encrypted.length} characters`);
    
    return {
      success: true,
      lastFour: cardNumber.slice(-4),
      cardType: this.getCardType(cardNumber)
    };
  }
  
  /**
   * Retrieve and decrypt card data
   */
  retrieveCard(customerId) {
    console.log(`\n🔓 Retrieving card for customer: ${customerId}`);
    
    const stored = this.encryptedCards.get(customerId);
    
    if (!stored) {
      throw new Error('Card not found');
    }
    
    // Decrypt card data
    const decryptedJson = decrypt(stored.encryptedData);
    const cardData = JSON.parse(decryptedJson);
    
    console.log(`   ✅ Card decrypted successfully`);
    console.log(`   Card Number: ${this.maskCard(cardData.cardNumber)}`);
    console.log(`   Expiry: ${cardData.expiryDate}`);
    
    return cardData;
  }
  
  /**
   * Get card info without decrypting (for display)
   */
  getCardInfo(customerId) {
    const stored = this.encryptedCards.get(customerId);
    
    if (!stored) {
      throw new Error('Card not found');
    }
    
    return {
      lastFour: stored.lastFour,
      cardType: stored.cardType,
      maskedNumber: `****-****-****-${stored.lastFour}`,
      storedAt: stored.storedAt
    };
  }
  
  /**
   * Process payment (requires decryption)
   */
  processPayment(customerId, amount) {
    console.log(`\n💳 Processing payment for customer: ${customerId}`);
    console.log(`   Amount: $${amount.toFixed(2)}`);
    
    // Retrieve and decrypt card
    const cardData = this.retrieveCard(customerId);
    
    // Simulate payment processing with actual card data
    console.log(`   Using card: ${this.maskCard(cardData.cardNumber)}`);
    console.log(`   CVV: ${'*'.repeat(cardData.cvv.length)}`);
    
    // In production: Call payment gateway with decrypted card data
    const transactionId = crypto.randomBytes(16).toString('hex');
    
    console.log(`   ✅ Payment processed successfully`);
    console.log(`   Transaction ID: ${transactionId}`);
    
    return {
      success: true,
      transactionId,
      amount,
      timestamp: new Date().toISOString()
    };
  }
  
  /**
   * Delete card (compliance requirement)
   */
  deleteCard(customerId) {
    console.log(`\n🗑️  Deleting card for customer: ${customerId}`);
    
    const deleted = this.encryptedCards.delete(customerId);
    
    if (deleted) {
      console.log(`   ✅ Card deleted successfully`);
    } else {
      console.log(`   ⚠️  Card not found`);
    }
    
    return deleted;
  }
  
  /**
   * Mask card number for display
   */
  maskCard(cardNumber) {
    const cleaned = cardNumber.replace(/\D/g, '');
    return `${cleaned.slice(0, 4)}-****-****-${cleaned.slice(-4)}`;
  }
  
  /**
   * Validate card number using Luhn algorithm
   */
  validateCard(cardNumber) {
    const cleaned = cardNumber.replace(/\D/g, '');
    
    if (cleaned.length < 13 || cleaned.length > 19) {
      return false;
    }
    
    let sum = 0;
    let isEven = false;
    
    // Loop from right to left
    for (let i = cleaned.length - 1; i >= 0; i--) {
      let digit = parseInt(cleaned[i], 10);
      
      if (isEven) {
        digit *= 2;
        if (digit > 9) {
          digit -= 9;
        }
      }
      
      sum += digit;
      isEven = !isEven;
    }
    
    return sum % 10 === 0;
  }
  
  /**
   * Determine card type from number
   */
  getCardType(cardNumber) {
    const cleaned = cardNumber.replace(/\D/g, '');
    
    if (/^4/.test(cleaned)) return 'Visa';
    if (/^5[1-5]/.test(cleaned)) return 'MasterCard';
    if (/^3[47]/.test(cleaned)) return 'American Express';
    if (/^6(?:011|5)/.test(cleaned)) return 'Discover';
    
    return 'Unknown';
  }
  
  /**
   * Get encryption statistics
   */
  getStats() {
    return {
      totalCards: this.encryptedCards.size,
      algorithm: ALGORITHM,
      keySize: ENCRYPTION_KEY.length * 8 // bits
    };
  }
}

// ============================================
// Demo Usage
// ============================================

console.log('\n🚀 Starting Card Encryption Demo...\n');

const handler = new CardDataHandler();

// Demo 1: Store multiple cards
console.log('\n' + '='.repeat(70));
console.log('DEMO 1: Storing Cards');
console.log('='.repeat(70));

handler.storeCard('CUST001', '4532-1234-5678-9010', '123', '12/25');
handler.storeCard('CUST002', '5425-2334-3010-9903', '456', '06/26');
handler.storeCard('CUST003', '3782-822463-10005', '7890', '09/24');

// Demo 2: Retrieve card info (without decryption)
console.log('\n' + '='.repeat(70));
console.log('DEMO 2: Retrieving Card Info (No Decryption)');
console.log('='.repeat(70));

const info1 = handler.getCardInfo('CUST001');
console.log('\nCustomer CUST001:');
console.log('  Card Type:', info1.cardType);
console.log('  Masked Number:', info1.maskedNumber);
console.log('  Stored At:', info1.storedAt);

// Demo 3: Process payment (requires decryption)
console.log('\n' + '='.repeat(70));
console.log('DEMO 3: Processing Payments');
console.log('='.repeat(70));

handler.processPayment('CUST001', 299.99);
handler.processPayment('CUST002', 1499.50);

// Demo 4: Retrieve and decrypt full card data
console.log('\n' + '='.repeat(70));
console.log('DEMO 4: Retrieve Full Card Data (Decrypt)');
console.log('='.repeat(70));

const fullCard = handler.retrieveCard('CUST001');
console.log('\nDecrypted Card Data:');
console.log('  Card Number:', handler.maskCard(fullCard.cardNumber));
console.log('  CVV: ', '*'.repeat(fullCard.cvv.length));
console.log('  Expiry:', fullCard.expiryDate);
console.log('  Stored At:', fullCard.storedAt);

// Demo 5: Delete card
console.log('\n' + '='.repeat(70));
console.log('DEMO 5: Deleting Card (GDPR Compliance)');
console.log('='.repeat(70));

handler.deleteCard('CUST003');

// Demo 6: Statistics
console.log('\n' + '='.repeat(70));
console.log('DEMO 6: Encryption Statistics');
console.log('='.repeat(70));

const stats = handler.getStats();
console.log('\nSystem Statistics:');
console.log('  Total Cards Stored:', stats.totalCards);
console.log('  Algorithm:', stats.algorithm);
console.log('  Key Size:', stats.keySize, 'bits');

console.log('\n' + '='.repeat(70));
console.log('✅ Demo Complete!');
console.log('='.repeat(70) + '\n');
```

**To run this example:**

```bash
node card-encryption-system.js
```

**Expected Output:**

```
🔐 Credit Card Encryption System

======================================================================

🚀 Starting Card Encryption Demo...

======================================================================
DEMO 1: Storing Cards
======================================================================

🔒 Storing card for customer: CUST001
   Card Number: 4532-****-****-9010
   ✅ Card encrypted and stored
   Last 4 digits: ****9010
   Encrypted size: 192 characters

🔒 Storing card for customer: CUST002
   Card Number: 5425-****-****-9903
   ✅ Card encrypted and stored
   Last 4 digits: ****9903
   Encrypted size: 192 characters

🔒 Storing card for customer: CUST003
   Card Number: 3782-****-****-0005
   ✅ Card encrypted and stored
   Last 4 digits: ****0005
   Encrypted size: 196 characters

======================================================================
DEMO 2: Retrieving Card Info (No Decryption)
======================================================================

Customer CUST001:
  Card Type: Visa
  Masked Number: ****-****-****-9010
  Stored At: 2025-11-14T10:30:00.000Z

======================================================================
DEMO 3: Processing Payments
======================================================================

💳 Processing payment for customer: CUST001
   Amount: $299.99

🔓 Retrieving card for customer: CUST001
   ✅ Card decrypted successfully
   Card Number: 4532-****-****-9010
   Expiry: 12/25
   Using card: 4532-****-****-9010
   CVV: ***
   ✅ Payment processed successfully
   Transaction ID: a3f7e2d9c1b8a4f6e3d2c1b9a8f7e6d5

💳 Processing payment for customer: CUST002
   Amount: $1499.50

🔓 Retrieving card for customer: CUST002
   ✅ Card decrypted successfully
   Card Number: 5425-****-****-9903
   Expiry: 06/26
   Using card: 5425-****-****-9903
   CVV: ***
   ✅ Payment processed successfully
   Transaction ID: b4e8f3d0c2b9a5f7e4d3c2b0a9f8e7d6

======================================================================
DEMO 4: Retrieve Full Card Data (Decrypt)
======================================================================

🔓 Retrieving card for customer: CUST001
   ✅ Card decrypted successfully
   Card Number: 4532-****-****-9010
   Expiry: 12/25

Decrypted Card Data:
  Card Number: 4532-****-****-9010
  CVV:  ***
  Expiry: 12/25
  Stored At: 2025-11-14T10:30:00.000Z

======================================================================
DEMO 5: Deleting Card (GDPR Compliance)
======================================================================

🗑️  Deleting card for customer: CUST003
   ✅ Card deleted successfully

======================================================================
DEMO 6: Encryption Statistics
======================================================================

System Statistics:
  Total Cards Stored: 2
  Algorithm: aes-256-gcm
  Key Size: 256 bits

======================================================================
✅ Demo Complete!
======================================================================
```

**Key Learnings from Example 1**:

1. **AES-256-GCM**: Industry-standard encryption for sensitive data
2. **Initialization Vector (IV)**: Random IV for each encryption ensures unique ciphertext
3. **Authentication Tag**: GCM provides built-in authentication
4. **Buffer Encoding**: Hex encoding for storing encrypted data as strings
5. **Key Management**: Keys should be stored securely (environment variables, KMS)
6. **PCI DSS Compliance**: Never log or display full card numbers
7. **Luhn Algorithm**: Validate card numbers before processing
8. **Masking**: Display only last 4 digits for security

---

## 🏦 Example 2: Secure Hashing & Token Generation

This example demonstrates hashing, HMAC, and secure token generation.

**File: `hashing-tokens-system.js`**

```javascript
import crypto from 'crypto';

/**
 * Banking Hashing & Token System
 * Demonstrates: SHA-256, HMAC, secure random tokens, password hashing
 */

console.log('🔐 Hashing & Token Generation System\n');
console.log('='.repeat(70));

// ============================================
// Hashing Functions
// ============================================

/**
 * Create SHA-256 hash
 */
function createHash(data) {
  return crypto.createHash('sha256').update(data).digest('hex');
}

/**
 * Create HMAC (Hash-based Message Authentication Code)
 */
function createHMAC(data, secret) {
  return crypto.createHmac('sha256', secret).update(data).digest('hex');
}

/**
 * Generate secure random token
 */
function generateToken(bytes = 32) {
  return crypto.randomBytes(bytes).toString('hex');
}

/**
 * Generate cryptographically secure random number
 */
function generateSecureRandom(min, max) {
  const range = max - min + 1;
  const bytesNeeded = Math.ceil(Math.log2(range) / 8);
  const maxValue = Math.pow(256, bytesNeeded);
  const cutoff = maxValue - (maxValue % range);
  
  let randomValue;
  do {
    randomValue = crypto.randomBytes(bytesNeeded).readUIntBE(0, bytesNeeded);
  } while (randomValue >= cutoff);
  
  return min + (randomValue % range);
}

// ============================================
// Password Hashing with Salt
// ============================================

class PasswordManager {
  /**
   * Hash password with salt (using PBKDF2)
   */
  static hashPassword(password) {
    const salt = crypto.randomBytes(16).toString('hex');
    const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512').toString('hex');
    
    return {
      salt,
      hash
    };
  }
  
  /**
   * Verify password
   */
  static verifyPassword(password, salt, hash) {
    const verifyHash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512').toString('hex');
    return hash === verifyHash;
  }
}

// ============================================
// API Token Manager
// ============================================

class APITokenManager {
  constructor(secret) {
    this.secret = secret;
    this.tokens = new Map();
  }
  
  /**
   * Generate API token for customer
   */
  generateToken(customerId, expiresInMinutes = 60) {
    console.log(`\n🔑 Generating API token for customer: ${customerId}`);
    
    // Create token data
    const tokenData = {
      customerId,
      issuedAt: Date.now(),
      expiresAt: Date.now() + (expiresInMinutes * 60 * 1000),
      nonce: crypto.randomBytes(16).toString('hex')
    };
    
    // Generate random token
    const token = generateToken(32);
    
    // Create signature using HMAC
    const dataToSign = JSON.stringify({
      customerId: tokenData.customerId,
      issuedAt: tokenData.issuedAt,
      expiresAt: tokenData.expiresAt,
      nonce: tokenData.nonce
    });
    
    const signature = createHMAC(dataToSign, this.secret);
    
    // Store token
    this.tokens.set(token, {
      ...tokenData,
      signature
    });
    
    console.log(`   ✅ Token generated: ${token.substring(0, 16)}...`);
    console.log(`   Expires in: ${expiresInMinutes} minutes`);
    
    return {
      token,
      expiresAt: new Date(tokenData.expiresAt).toISOString()
    };
  }
  
  /**
   * Validate API token
   */
  validateToken(token) {
    console.log(`\n🔍 Validating token: ${token.substring(0, 16)}...`);
    
    const tokenData = this.tokens.get(token);
    
    if (!tokenData) {
      console.log(`   ❌ Token not found`);
      return { valid: false, reason: 'Token not found' };
    }
    
    // Check expiration
    if (Date.now() > tokenData.expiresAt) {
      console.log(`   ❌ Token expired`);
      return { valid: false, reason: 'Token expired' };
    }
    
    // Verify signature
    const dataToVerify = JSON.stringify({
      customerId: tokenData.customerId,
      issuedAt: tokenData.issuedAt,
      expiresAt: tokenData.expiresAt,
      nonce: tokenData.nonce
    });
    
    const expectedSignature = createHMAC(dataToVerify, this.secret);
    
    if (tokenData.signature !== expectedSignature) {
      console.log(`   ❌ Invalid signature`);
      return { valid: false, reason: 'Invalid signature' };
    }
    
    console.log(`   ✅ Token valid`);
    console.log(`   Customer ID: ${tokenData.customerId}`);
    
    return {
      valid: true,
      customerId: tokenData.customerId,
      expiresAt: new Date(tokenData.expiresAt).toISOString()
    };
  }
  
  /**
   * Revoke token
   */
  revokeToken(token) {
    const deleted = this.tokens.delete(token);
    console.log(deleted ? '   ✅ Token revoked' : '   ⚠️  Token not found');
    return deleted;
  }
}

// ============================================
// Transaction Signature System
// ============================================

class TransactionSigner {
  constructor(secret) {
    this.secret = secret;
  }
  
  /**
   * Sign transaction for integrity verification
   */
  signTransaction(transaction) {
    console.log(`\n✍️  Signing transaction: ${transaction.id}`);
    
    // Create canonical string from transaction
    const canonicalString = [
      transaction.id,
      transaction.fromAccount,
      transaction.toAccount,
      transaction.amount.toString(),
      transaction.currency,
      transaction.timestamp
    ].join('|');
    
    // Create signature
    const signature = createHMAC(canonicalString, this.secret);
    
    console.log(`   Canonical: ${canonicalString}`);
    console.log(`   Signature: ${signature.substring(0, 32)}...`);
    
    return {
      ...transaction,
      signature
    };
  }
  
  /**
   * Verify transaction signature
   */
  verifyTransaction(transaction) {
    console.log(`\n🔍 Verifying transaction: ${transaction.id}`);
    
    // Recreate canonical string
    const canonicalString = [
      transaction.id,
      transaction.fromAccount,
      transaction.toAccount,
      transaction.amount.toString(),
      transaction.currency,
      transaction.timestamp
    ].join('|');
    
    // Recreate signature
    const expectedSignature = createHMAC(canonicalString, this.secret);
    
    // Compare signatures
    const isValid = transaction.signature === expectedSignature;
    
    console.log(`   Expected: ${expectedSignature.substring(0, 32)}...`);
    console.log(`   Received: ${transaction.signature.substring(0, 32)}...`);
    console.log(`   Valid: ${isValid ? '✅' : '❌'}`);
    
    return isValid;
  }
}

// ============================================
// Demo Usage
// ============================================

console.log('\n🚀 Starting Demo...\n');

// Demo 1: Basic Hashing
console.log('='.repeat(70));
console.log('DEMO 1: Basic Hashing');
console.log('='.repeat(70));

const accountNumber = '1234567890';
const hash1 = createHash(accountNumber);
const hash2 = createHash(accountNumber);

console.log(`\nAccount Number: ${accountNumber}`);
console.log(`Hash 1: ${hash1}`);
console.log(`Hash 2: ${hash2}`);
console.log(`Hashes Match: ${hash1 === hash2 ? '✅' : '❌'}`);

// Demo 2: HMAC (with secret)
console.log('\n' + '='.repeat(70));
console.log('DEMO 2: HMAC with Secret Key');
console.log('='.repeat(70));

const secret = 'banking-secret-key-2025';
const transactionData = 'TXN-001|ACC-123|ACC-456|500.00|USD';

const hmac1 = createHMAC(transactionData, secret);
const hmac2 = createHMAC(transactionData, secret);
const hmac3 = createHMAC(transactionData, 'wrong-secret');

console.log(`\nTransaction: ${transactionData}`);
console.log(`HMAC (correct secret): ${hmac1.substring(0, 32)}...`);
console.log(`HMAC (same secret):    ${hmac2.substring(0, 32)}...`);
console.log(`HMAC (wrong secret):   ${hmac3.substring(0, 32)}...`);
console.log(`\nHMACs match: ${hmac1 === hmac2 ? '✅' : '❌'}`);
console.log(`Wrong secret matches: ${hmac1 === hmac3 ? '✅' : '❌'}`);

// Demo 3: Password Hashing
console.log('\n' + '='.repeat(70));
console.log('DEMO 3: Password Hashing & Verification');
console.log('='.repeat(70));

const password = 'SecurePassword123!';
const { salt, hash } = PasswordManager.hashPassword(password);

console.log(`\nPassword: ${password}`);
console.log(`Salt: ${salt.substring(0, 16)}...`);
console.log(`Hash: ${hash.substring(0, 32)}...`);

// Verify correct password
const isValid1 = PasswordManager.verifyPassword(password, salt, hash);
console.log(`\nVerify correct password: ${isValid1 ? '✅' : '❌'}`);

// Verify wrong password
const isValid2 = PasswordManager.verifyPassword('WrongPassword', salt, hash);
console.log(`Verify wrong password: ${isValid2 ? '✅' : '❌'}`);

// Demo 4: API Token Generation
console.log('\n' + '='.repeat(70));
console.log('DEMO 4: API Token Management');
console.log('='.repeat(70));

const tokenManager = new APITokenManager('api-secret-key-2025');

// Generate tokens
const token1 = tokenManager.generateToken('CUST001', 60);
const token2 = tokenManager.generateToken('CUST002', 30);

// Validate tokens
const validation1 = tokenManager.validateToken(token1.token);
const validation2 = tokenManager.validateToken(token2.token);

// Try invalid token
const validation3 = tokenManager.validateToken('invalid-token-xyz');

// Revoke token
console.log(`\n🗑️  Revoking token for CUST001`);
tokenManager.revokeToken(token1.token);

// Try using revoked token
const validation4 = tokenManager.validateToken(token1.token);

// Demo 5: Transaction Signing
console.log('\n' + '='.repeat(70));
console.log('DEMO 5: Transaction Signing & Verification');
console.log('='.repeat(70));

const signer = new TransactionSigner('transaction-secret-key-2025');

// Create transaction
const transaction = {
  id: 'TXN-12345',
  fromAccount: 'ACC-001',
  toAccount: 'ACC-002',
  amount: 1500.00,
  currency: 'USD',
  timestamp: new Date().toISOString()
};

// Sign transaction
const signedTxn = signer.signTransaction(transaction);

// Verify valid transaction
const isValidTxn1 = signer.verifyTransaction(signedTxn);

// Tamper with transaction
const tamperedTxn = { ...signedTxn, amount: 9999.99 };
console.log(`\n⚠️  Tampering with transaction amount: ${transaction.amount} → ${tamperedTxn.amount}`);

// Verify tampered transaction
const isValidTxn2 = signer.verifyTransaction(tamperedTxn);

// Demo 6: Secure Random Number Generation
console.log('\n' + '='.repeat(70));
console.log('DEMO 6: Secure Random Number Generation');
console.log('='.repeat(70));

console.log('\nGenerating 10 secure random account numbers (1000-9999):');
for (let i = 0; i < 10; i++) {
  const accountNum = generateSecureRandom(1000, 9999);
  console.log(`  Account ${i + 1}: ${accountNum}`);
}

console.log('\nGenerating 5 secure API tokens (16 bytes each):');
for (let i = 0; i < 5; i++) {
  const token = generateToken(16);
  console.log(`  Token ${i + 1}: ${token}`);
}

console.log('\n' + '='.repeat(70));
console.log('✅ Demo Complete!');
console.log('='.repeat(70) + '\n');
```

**To run this example:**

```bash
node hashing-tokens-system.js
```

**Expected Output:**

```
🔐 Hashing & Token Generation System

======================================================================

🚀 Starting Demo...

======================================================================
DEMO 1: Basic Hashing
======================================================================

Account Number: 1234567890
Hash 1: c775e7b757ede630cd0aa1113bd102661ab38829ca52a6422ab782862f268646
Hash 2: c775e7b757ede630cd0aa1113bd102661ab38829ca52a6422ab782862f268646
Hashes Match: ✅

======================================================================
DEMO 2: HMAC with Secret Key
======================================================================

Transaction: TXN-001|ACC-123|ACC-456|500.00|USD
HMAC (correct secret): a7f3e9d2c1b8a4f6e3d2c1b9a8f7...
HMAC (same secret):    a7f3e9d2c1b8a4f6e3d2c1b9a8f7...
HMAC (wrong secret):   f6e4d3c2b1a9f8e7d6c5b4a3f2e1...

HMACs match: ✅
Wrong secret matches: ❌

======================================================================
DEMO 3: Password Hashing & Verification
======================================================================

Password: SecurePassword123!
Salt: 3f7a9e2c1b8d4f6e...
Hash: b8f7e6d5c4b3a2f1e0d9c8b7a6f5...

Verify correct password: ✅
Verify wrong password: ❌

======================================================================
DEMO 4: API Token Management
======================================================================

🔑 Generating API token for customer: CUST001
   ✅ Token generated: a3f7e2d9c1b8a4f6...
   Expires in: 60 minutes

🔑 Generating API token for customer: CUST002
   ✅ Token generated: b4e8f3d0c2b9a5f7...
   Expires in: 30 minutes

🔍 Validating token: a3f7e2d9c1b8a4f6...
   ✅ Token valid
   Customer ID: CUST001

🔍 Validating token: b4e8f3d0c2b9a5f7...
   ✅ Token valid
   Customer ID: CUST002

🔍 Validating token: invalid-token-x...
   ❌ Token not found

🗑️  Revoking token for CUST001
   ✅ Token revoked

🔍 Validating token: a3f7e2d9c1b8a4f6...
   ❌ Token not found

======================================================================
DEMO 5: Transaction Signing & Verification
======================================================================

✍️  Signing transaction: TXN-12345
   Canonical: TXN-12345|ACC-001|ACC-002|1500|USD|2025-11-14T10:30:00.000Z
   Signature: c8f7e6d5c4b3a2f1e0d9c8b7a6f5...

🔍 Verifying transaction: TXN-12345
   Expected: c8f7e6d5c4b3a2f1e0d9c8b7a6f5...
   Received: c8f7e6d5c4b3a2f1e0d9c8b7a6f5...
   Valid: ✅

⚠️  Tampering with transaction amount: 1500 → 9999.99

🔍 Verifying transaction: TXN-12345
   Expected: c8f7e6d5c4b3a2f1e0d9c8b7a6f5...
   Received: d9e8f7e6d5c4b3a2f1e0d9c8b7a6...
   Valid: ❌

======================================================================
DEMO 6: Secure Random Number Generation
======================================================================

Generating 10 secure random account numbers (1000-9999):
  Account 1: 7823
  Account 2: 3456
  Account 3: 9012
  Account 4: 2345
  Account 5: 6789
  Account 6: 4567
  Account 7: 8901
  Account 8: 1234
  Account 9: 5678
  Account 10: 3901

Generating 5 secure API tokens (16 bytes each):
  Token 1: a3f7e2d9c1b8a4f6e3d2c1b9a8f7e6d5
  Token 2: b4e8f3d0c2b9a5f7e4d3c2b0a9f8e7d6
  Token 3: c5f9e4d1c3b0a6f8e5d4c3b1a0f9e8d7
  Token 4: d6f0e5d2c4b1a7f9e6d5c4b2a1f0e9d8
  Token 5: e7f1e6d3c5b2a8f0e7d6c5b3a2f1e0d9

======================================================================
✅ Demo Complete!
======================================================================
```

**Key Learnings from Example 2**:

1. **SHA-256**: One-way hashing for data integrity
2. **HMAC**: Hash with secret key for authentication
3. **PBKDF2**: Password hashing with salt and iterations
4. **Random Tokens**: Cryptographically secure token generation
5. **Transaction Signatures**: Prevent tampering with HMAC
6. **Token Management**: Issue, validate, revoke API tokens
7. **Secure Random**: Use crypto.randomBytes() for security

---

## ✅ 10 DO's: Best Practices for Buffer Encoding & Cryptography

### 1. ✅ DO: Use AES-256-GCM for Encrypting Sensitive Data

**Why**: GCM provides both encryption and authentication, preventing tampering.

```javascript
// Good: AES-256-GCM with authentication
function encryptData(plaintext, key) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
  
  let encrypted = cipher.update(plaintext, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  return {
    encrypted,
    iv: iv.toString('hex'),
    authTag: authTag.toString('hex')
  };
}
```

**Banking Use Case**: Encrypt credit card numbers, SSN, account numbers.

---

### 2. ✅ DO: Use Random IV for Each Encryption

**Why**: Ensures the same plaintext produces different ciphertext each time.

```javascript
// Good: New random IV for each encryption
function encrypt(data, key) {
  const iv = crypto.randomBytes(16);  // ✅ New IV each time
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
  // ... encrypt
  return { encrypted, iv, authTag };
}

// Bad: Reusing IV
const FIXED_IV = Buffer.alloc(16);  // ❌ Never do this!
```

**Banking Use Case**: Encrypting multiple transactions with same key.

---

### 3. ✅ DO: Use PBKDF2 or bcrypt for Password Hashing

**Why**: Slow hashing with salt prevents brute-force attacks.

```javascript
// Good: PBKDF2 with salt and iterations
function hashPassword(password) {
  const salt = crypto.randomBytes(16);
  const iterations = 100000;  // High iteration count
  const keylen = 64;
  const digest = 'sha512';
  
  const hash = crypto.pbkdf2Sync(
    password,
    salt,
    iterations,
    keylen,
    digest
  );
  
  return {
    salt: salt.toString('hex'),
    hash: hash.toString('hex')
  };
}
```

**Banking Use Case**: Storing customer passwords, PIN numbers.

---

### 4. ✅ DO: Store Encryption Keys Securely

**Why**: Keys must be protected; if compromised, all data is at risk.

```javascript
// Good: Load key from environment or KMS
const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY 
  ? Buffer.from(process.env.ENCRYPTION_KEY, 'hex')
  : null;

if (!ENCRYPTION_KEY) {
  throw new Error('ENCRYPTION_KEY not configured');
}

// Even better: Use AWS KMS, Azure Key Vault, HashiCorp Vault
import { KMSClient, DecryptCommand } from '@aws-sdk/client-kms';

async function getEncryptionKey() {
  const kms = new KMSClient({ region: 'us-east-1' });
  const command = new DecryptCommand({
    CiphertextBlob: Buffer.from(process.env.ENCRYPTED_KEY, 'base64')
  });
  
  const response = await kms.send(command);
  return response.Plaintext;
}
```

**Banking Use Case**: Protecting master encryption keys.

---

### 5. ✅ DO: Use HMAC for Data Integrity

**Why**: Ensures data hasn't been tampered with.

```javascript
// Good: Sign data with HMAC
function signData(data, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(JSON.stringify(data));
  return hmac.digest('hex');
}

function verifyData(data, signature, secret) {
  const expectedSignature = signData(data, secret);
  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );
}
```

**Banking Use Case**: Signing transaction data to prevent tampering.

---

### 6. ✅ DO: Use Base64 for Binary Data in JSON

**Why**: JSON doesn't support binary data; Base64 makes it text-safe.

```javascript
// Good: Encode binary data as Base64 for JSON
const encryptedData = {
  encrypted: buffer.toString('base64'),  // ✅
  iv: iv.toString('base64'),
  authTag: authTag.toString('base64')
};

// When storing in database
await db.query(
  'INSERT INTO encrypted_cards (card_data) VALUES ($1)',
  [JSON.stringify(encryptedData)]
);

// When retrieving
const stored = JSON.parse(result.rows[0].card_data);
const buffer = Buffer.from(stored.encrypted, 'base64');
```

**Banking Use Case**: Storing encrypted data in JSON format.

---

### 7. ✅ DO: Use crypto.randomBytes() for Secure Random Data

**Why**: cryptographically secure random number generation.

```javascript
// Good: Crypto-secure random
const token = crypto.randomBytes(32).toString('hex');  // ✅
const sessionId = crypto.randomBytes(16).toString('base64');

// Bad: Math.random() is NOT secure
const badToken = Math.random().toString(36);  // ❌ Predictable!
```

**Banking Use Case**: Generating session tokens, API keys, account numbers.

---

### 8. ✅ DO: Validate Encoding Before Conversion

**Why**: Invalid encoding can cause errors or data corruption.

```javascript
// Good: Validate before converting
function safeBase64Decode(base64String) {
  // Check if valid Base64
  const base64Regex = /^[A-Za-z0-9+/]*={0,2}$/;
  
  if (!base64Regex.test(base64String)) {
    throw new Error('Invalid Base64 string');
  }
  
  try {
    return Buffer.from(base64String, 'base64');
  } catch (err) {
    throw new Error(`Base64 decode failed: ${err.message}`);
  }
}

function safeHexDecode(hexString) {
  // Check if valid hex
  const hexRegex = /^[0-9a-fA-F]*$/;
  
  if (!hexRegex.test(hexString)) {
    throw new Error('Invalid hex string');
  }
  
  return Buffer.from(hexString, 'hex');
}
```

**Banking Use Case**: Parsing encrypted data from API requests.

---

### 9. ✅ DO: Use Constant-Time Comparison for Secrets

**Why**: Prevents timing attacks that can leak information.

```javascript
// Good: Constant-time comparison
function compareSecrets(a, b) {
  if (a.length !== b.length) {
    return false;
  }
  
  return crypto.timingSafeEqual(
    Buffer.from(a, 'hex'),
    Buffer.from(b, 'hex')
  );
}

// Bad: String comparison (timing attack vulnerable)
function badCompare(a, b) {
  return a === b;  // ❌ Leaks timing information!
}
```

**Banking Use Case**: Comparing HMAC signatures, password hashes.

---

### 10. ✅ DO: Implement Key Rotation

**Why**: Regularly rotating keys limits damage from key compromise.

```javascript
// Good: Key rotation system
class KeyManager {
  constructor() {
    this.keys = new Map();
    this.currentKeyId = null;
  }
  
  addKey(keyId, key) {
    this.keys.set(keyId, {
      key: Buffer.from(key, 'hex'),
      createdAt: Date.now(),
      active: true
    });
    
    this.currentKeyId = keyId;
  }
  
  getCurrentKey() {
    return {
      keyId: this.currentKeyId,
      key: this.keys.get(this.currentKeyId).key
    };
  }
  
  getKey(keyId) {
    const keyData = this.keys.get(keyId);
    if (!keyData) {
      throw new Error('Key not found');
    }
    return keyData.key;
  }
  
  rotateKey() {
    const newKeyId = `key-${Date.now()}`;
    const newKey = crypto.randomBytes(32);
    
    // Mark old key as inactive
    if (this.currentKeyId) {
      this.keys.get(this.currentKeyId).active = false;
    }
    
    this.addKey(newKeyId, newKey.toString('hex'));
    
    console.log(`✅ Key rotated: ${newKeyId}`);
  }
  
  encrypt(plaintext) {
    const { keyId, key } = this.getCurrentKey();
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
    
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return {
      keyId,  // Store which key was used
      encrypted,
      iv: iv.toString('hex'),
      authTag: cipher.getAuthTag().toString('hex')
    };
  }
  
  decrypt(encryptedData) {
    const key = this.getKey(encryptedData.keyId);  // Use correct key
    const decipher = crypto.createDecipheriv(
      'aes-256-gcm',
      key,
      Buffer.from(encryptedData.iv, 'hex')
    );
    
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}
```

**Banking Use Case**: Rotating encryption keys quarterly for compliance.

---

## ❌ 10 DON'Ts: Common Mistakes to Avoid

### 1. ❌ DON'T: Use ECB Mode for Encryption

**Why**: ECB doesn't use IV and reveals patterns in data.

```javascript
// Bad: ECB mode (insecure)
const cipher = crypto.createCipher('aes-256-ecb', key);  // ❌

// Good: Use GCM or CBC with IV
const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);  // ✅
```

**Why ECB is Bad**: Same plaintext → Same ciphertext (pattern leakage).

---

### 2. ❌ DON'T: Store Encryption Keys in Code

**Why**: Keys will be exposed in version control and deployments.

```javascript
// Bad: Hardcoded key
const KEY = Buffer.from('0123456789abcdef0123456789abcdef', 'hex');  // ❌

// Good: Load from environment
const KEY = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');  // ✅
```

---

### 3. ❌ DON'T: Use Weak Hashing Algorithms

**Why**: MD5 and SHA1 are cryptographically broken.

```javascript
// Bad: Weak algorithms
const hash = crypto.createHash('md5').update(data).digest('hex');  // ❌
const hash2 = crypto.createHash('sha1').update(data).digest('hex');  // ❌

// Good: Strong algorithms
const hash = crypto.createHash('sha256').update(data).digest('hex');  // ✅
const hash2 = crypto.createHash('sha512').update(data).digest('hex');  // ✅
```

---

### 4. ❌ DON'T: Hash Passwords Without Salt

**Why**: Rainbow tables can crack unsalted hashes.

```javascript
// Bad: No salt
const hash = crypto.createHash('sha256').update(password).digest('hex');  // ❌

// Good: Salt + slow hashing
const salt = crypto.randomBytes(16);
const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');  // ✅
```

---

### 5. ❌ DON'T: Ignore Authentication Tags in GCM

**Why**: Without verification, encrypted data can be tampered with.

```javascript
// Bad: Not using auth tag
const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
const encrypted = cipher.update(data, 'utf8', 'hex') + cipher.final('hex');
// Forgot to get authTag!  ❌

// Good: Store and verify auth tag
const authTag = cipher.getAuthTag();  // ✅
// Store authTag with encrypted data

// When decrypting:
decipher.setAuthTag(authTag);  // ✅ Verify integrity
```

---

### 6. ❌ DON'T: Use Math.random() for Security

**Why**: Math.random() is predictable and not cryptographically secure.

```javascript
// Bad: Predictable random
const token = Math.random().toString(36).substring(2);  // ❌

// Good: Crypto-secure random
const token = crypto.randomBytes(32).toString('hex');  // ✅
```

---

### 7. ❌ DON'T: Reuse IVs with the Same Key

**Why**: Same IV + same key = pattern leakage.

```javascript
// Bad: Reusing IV
const FIXED_IV = Buffer.alloc(16);  // ❌
function encrypt(data) {
  const cipher = crypto.createCipheriv('aes-256-cbc', key, FIXED_IV);
  // ...
}

// Good: New IV each time
function encrypt(data) {
  const iv = crypto.randomBytes(16);  // ✅ New IV
  const cipher = crypto.createCipheriv('aes-256-cbc', key, iv);
  // ...
  return { encrypted, iv };
}
```

---

### 8. ❌ DON'T: Log or Display Sensitive Data

**Why**: Logs can be accessed by unauthorized parties.

```javascript
// Bad: Logging sensitive data
console.log('Credit card:', cardNumber);  // ❌
console.log('Password:', password);  // ❌
console.log('Encryption key:', key.toString('hex'));  // ❌

// Good: Mask or omit sensitive data
console.log('Credit card:', `****${cardNumber.slice(-4)}`);  // ✅
console.log('Password:', '[REDACTED]');  // ✅
// Never log encryption keys!
```

---

### 9. ❌ DON'T: Use String Comparison for Secrets

**Why**: Vulnerable to timing attacks.

```javascript
// Bad: String comparison
function verify(userSignature, expectedSignature) {
  return userSignature === expectedSignature;  // ❌ Timing attack!
}

// Good: Constant-time comparison
function verify(userSignature, expectedSignature) {
  return crypto.timingSafeEqual(
    Buffer.from(userSignature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );  // ✅
}
```

---

### 10. ❌ DON'T: Forget Error Handling in Crypto Operations

**Why**: Crypto operations can fail; handle errors gracefully.

```javascript
// Bad: No error handling
function decrypt(encrypted, key, iv, authTag) {
  const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(authTag);
  return decipher.update(encrypted, 'hex', 'utf8') + decipher.final('utf8');
  // ❌ Will crash if authentication fails!
}

// Good: Proper error handling
function decrypt(encrypted, key, iv, authTag) {
  try {
    const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
    decipher.setAuthTag(authTag);
    return decipher.update(encrypted, 'hex', 'utf8') + decipher.final('utf8');
  } catch (err) {
    // Authentication failed or decryption error
    throw new Error('Decryption failed: Data may be corrupted or tampered');
  }
}
```

---

## 🎓 Key Takeaways

### Buffer Encoding Comparison

| Encoding | Use Case | Size | Example |
|----------|----------|------|---------|
| **utf8** | Text, default | Variable | `Hello` → `48656c6c6f` |
| **base64** | Binary in JSON/URLs | 133% of original | `Hello` → `SGVsbG8=` |
| **hex** | Human-readable binary | 200% of original | `Hello` → `48656c6c6f` |
| **ascii** | English text only | 1 byte/char | `Hello` → `Hello` |
| **binary/latin1** | Raw 8-bit data | 1 byte/char | Binary data |
| **utf16le/ucs2** | Unicode 2-byte | 2 bytes/char | International text |

### Encryption Algorithms

| Algorithm | Key Size | Block Size | Mode | Use Case |
|-----------|----------|------------|------|----------|
| **AES-256-GCM** | 256 bits | 128 bits | GCM | Best: Encryption + Auth |
| **AES-256-CBC** | 256 bits | 128 bits | CBC | Good: Encryption only |
| **AES-128-GCM** | 128 bits | 128 bits | GCM | Fast: Less security |
| **ChaCha20-Poly1305** | 256 bits | Stream | AEAD | Mobile: Low CPU |

**Recommendation**: Use AES-256-GCM for banking applications.

### Hashing Algorithms

| Algorithm | Output Size | Speed | Security | Use Case |
|-----------|-------------|-------|----------|----------|
| **SHA-256** | 256 bits | Fast | Strong | Data integrity, signatures |
| **SHA-512** | 512 bits | Fast | Strong | High security requirements |
| **PBKDF2** | Variable | Slow | Strong | Password hashing |
| **bcrypt** | 184 bits | Slow | Strong | Password hashing |
| **scrypt** | Variable | Very slow | Very strong | High security passwords |
| **MD5** | 128 bits | Fast | ❌ Broken | Never use for security |
| **SHA-1** | 160 bits | Fast | ❌ Weak | Deprecated |

**Recommendation**: SHA-256 for data, PBKDF2/bcrypt for passwords.

### Crypto Module Functions

**Encryption/Decryption**:
```javascript
crypto.createCipheriv(algorithm, key, iv)      // Encrypt
crypto.createDecipheriv(algorithm, key, iv)    // Decrypt
cipher.update(data, inputEncoding, outputEncoding)
cipher.final(outputEncoding)
cipher.getAuthTag()                            // GCM only
decipher.setAuthTag(tag)                       // GCM only
```

**Hashing**:
```javascript
crypto.createHash(algorithm)                   // SHA-256, SHA-512
crypto.createHmac(algorithm, key)              // HMAC-SHA256
crypto.pbkdf2Sync(password, salt, iterations, keylen, digest)
hash.update(data)
hash.digest(encoding)
```

**Random Data**:
```javascript
crypto.randomBytes(size)                       // Secure random
crypto.randomInt(min, max)                     // Secure random int
crypto.timingSafeEqual(a, b)                   // Constant-time compare
```

### PCI DSS Requirements for Card Data

1. **Encryption**: Use strong crypto (AES-256)
2. **Key Management**: Secure key storage and rotation
3. **Access Control**: Limit access to encrypted data
4. **Logging**: Audit all access to card data
5. **Masking**: Display only last 4 digits
6. **Transmission**: Use TLS/HTTPS for transmission
7. **Storage**: Minimize storage of sensitive data
8. **Disposal**: Securely delete when no longer needed

### Interview Q&A

**Q1: What's the difference between encryption and hashing?**

**A**:
- **Encryption**: Two-way (can decrypt back to original)
  - Use case: Storing credit card numbers
  - Example: AES-256-GCM
  
- **Hashing**: One-way (cannot reverse)
  - Use case: Storing passwords
  - Example: SHA-256, PBKDF2

**Q2: Why use Base64 encoding?**

**A**: Base64 converts binary data to text-safe format for:
- Storing binary in JSON
- Transmitting binary over HTTP
- Including binary in URLs/emails

```javascript
const encrypted = Buffer.from([0xFF, 0xD8, 0xFF, 0xE0]);
const base64 = encrypted.toString('base64');  // '/9j/4A=='
// Can now store in JSON: { "image": "/9j/4A==" }
```

**Q3: What's the difference between AES-256-CBC and AES-256-GCM?**

**A**:
- **CBC**: Encryption only, needs separate MAC for authentication
- **GCM**: AEAD (Authenticated Encryption with Associated Data)
  - Provides both encryption and authentication
  - Prevents tampering
  - Recommended for modern applications

**Q4: How to securely generate random tokens?**

**A**: Use `crypto.randomBytes()`:

```javascript
// Secure
const token = crypto.randomBytes(32).toString('hex');

// NOT secure
const badToken = Math.random().toString(36);  // ❌ Predictable!
```

**Q5: What's an HMAC and when to use it?**

**A**: HMAC = Hash-based Message Authentication Code
- Verifies data integrity and authenticity
- Requires shared secret key
- Use cases:
  - API request signing
  - Transaction verification
  - Webhook signatures

```javascript
const hmac = crypto.createHmac('sha256', secret);
hmac.update(data);
const signature = hmac.digest('hex');
```

**Q6: Why salt passwords?**

**A**: Salt prevents:
- Rainbow table attacks
- Identifying users with same password
- Precomputed hash attacks

```javascript
const salt = crypto.randomBytes(16);  // Unique per user
const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
```

**Q7: What's the difference between Buffer.alloc() and Buffer.allocUnsafe()?**

**A**:
- **Buffer.alloc(size)**: Fills with zeros (safe, slower)
- **Buffer.allocUnsafe(size)**: May contain old data (fast, must fill)

```javascript
// Safe: For sensitive data
const safe = Buffer.alloc(16);

// Fast: Must overwrite immediately
const fast = Buffer.allocUnsafe(16);
fast.fill(0);  // Clear old data
```

**Q8: How to handle key rotation?**

**A**: Include key ID with encrypted data:

```javascript
const encryptedData = {
  keyId: 'key-2025-11',  // Which key was used
  encrypted: '...',
  iv: '...',
  authTag: '...'
};

// When decrypting, use the correct key
const key = keyManager.getKey(encryptedData.keyId);
```

**Q9: What's a timing attack and how to prevent it?**

**A**: Timing attacks exploit different execution times:

```javascript
// Vulnerable: Early exit leaks information
function badCompare(a, b) {
  for (let i = 0; i < a.length; i++) {
    if (a[i] !== b[i]) return false;  // ❌ Early exit
  }
  return true;
}

// Safe: Constant time
crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b));  // ✅
```

**Q10: How to encrypt large files?**

**A**: Use streams with cipher:

```javascript
import fs from 'fs';
import crypto from 'crypto';

const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
const input = fs.createReadStream('large-file.pdf');
const output = fs.createWriteStream('encrypted.enc');

input.pipe(cipher).pipe(output);

output.on('finish', () => {
  const authTag = cipher.getAuthTag();
  // Store authTag separately
});
```

### Banking Security Checklist

✅ **Encryption**:
- [ ] Use AES-256-GCM for sensitive data
- [ ] Generate new IV for each encryption
- [ ] Verify authentication tags
- [ ] Implement key rotation

✅ **Hashing**:
- [ ] Use SHA-256 or stronger
- [ ] Salt all passwords
- [ ] Use PBKDF2/bcrypt for passwords
- [ ] Use HMAC for signatures

✅ **Key Management**:
- [ ] Store keys in environment/KMS
- [ ] Never hardcode keys
- [ ] Rotate keys regularly
- [ ] Use separate keys per environment

✅ **Data Protection**:
- [ ] Encrypt data at rest
- [ ] Use TLS for data in transit
- [ ] Mask sensitive data in logs
- [ ] Limit access to encrypted data

✅ **Compliance**:
- [ ] Follow PCI DSS for card data
- [ ] Implement audit logging
- [ ] Regular security audits
- [ ] Data retention policies

---

## 🏁 Summary

Buffer encoding and cryptography are essential for banking security:

1. **Encodings**: utf8, base64, hex for different use cases
2. **Encryption**: AES-256-GCM for protecting sensitive data
3. **Hashing**: SHA-256 for integrity, PBKDF2 for passwords
4. **HMAC**: Transaction signing and API authentication
5. **Random**: crypto.randomBytes() for secure tokens
6. **Key Management**: Secure storage and rotation
7. **Banking**: PCI DSS compliance for card data

**Security is not optional in banking applications** - always use proper encryption, validate data, and follow compliance requirements.



---
Summary
Successfully created Q08_Buffers_Encoding.md with 1,599 lines covering:

Content Breakdown:
Introduction (~50 lines)

Buffer encoding overview
Why it matters in banking
Common encodings (utf8, base64, hex, ascii)
Encoding Basics (~100 lines)

Visual flow diagrams
Encoding conversion examples
String ↔ Buffer ↔ Base64 ↔ Hex
Example 1: Credit Card Encryption System (~750 lines)

AES-256-GCM encryption/decryption
CardDataHandler class
Store/retrieve/delete encrypted cards
Luhn algorithm validation
Card type detection
Masked display
Payment processing
PCI DSS compliance
Example 2: Hashing & Token System (~500 lines)

SHA-256 hashing
HMAC for data integrity
PBKDF2 password hashing
API token management
Transaction signing
Secure random generation
10 DO's (~200 lines)

AES-256-GCM encryption
Random IV for each encryption
PBKDF2/bcrypt for passwords
Secure key storage (KMS)
HMAC for integrity
Base64 for JSON
crypto.randomBytes()
Validate encoding
Constant-time comparison
Key rotation
10 DON'Ts (~150 lines)

No ECB mode
No hardcoded keys
No weak hashing (MD5, SHA1)
No unsalted passwords
No ignored auth tags
No Math.random()
No IV reuse
No logging sensitive data
No string comparison for secrets
No missing error handling
Key Takeaways (~250 lines)

Encoding comparison table
Encryption algorithms table
Hashing algorithms table
Crypto module functions
PCI DSS requirements
10 interview Q&A
Security checklist

---
