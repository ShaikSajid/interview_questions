# Q28: Security - Data Protection

## 📋 Summary
This question covers **data encryption and secure communication** in Node.js applications, including AES symmetric encryption, RSA asymmetric encryption, and HTTPS/TLS setup. You'll learn how to protect sensitive banking data at rest and in transit, implement end-to-end encryption, and secure API communications for financial systems.

**What You'll Learn**:
- AES-256 symmetric encryption for data at rest
- RSA asymmetric encryption for key exchange
- HTTPS/TLS certificate setup and configuration
- Encrypting sensitive banking data (PII, account numbers, SSN)
- Key management and rotation strategies
- Secure data transmission patterns
- Compliance with PCI-DSS and banking regulations
- Practical encryption implementation patterns

---

## 🎯 Comprehensive Explanation

### Encryption Fundamentals

#### Encryption vs Hashing

**Encryption** (Reversible):
- Data can be decrypted with key
- Used for: Credit cards, SSN, account numbers
- Algorithms: AES, RSA, ChaCha20

**Hashing** (One-way):
- Cannot be reversed
- Used for: Passwords, data integrity
- Algorithms: bcrypt, argon2, SHA-256

#### Symmetric vs Asymmetric Encryption

**Symmetric (Same Key)**:
```
Plaintext → Encrypt (Key) → Ciphertext → Decrypt (Same Key) → Plaintext
```
- Fast, efficient for large data
- Key distribution challenge
- Examples: AES, ChaCha20

**Asymmetric (Public/Private Key Pair)**:
```
Plaintext → Encrypt (Public Key) → Ciphertext → Decrypt (Private Key) → Plaintext
```
- Slower, used for small data
- Secure key exchange
- Examples: RSA, ECC

### AES (Advanced Encryption Standard)

**AES-256-GCM** (Galois/Counter Mode):
- Symmetric encryption (same key encrypts/decrypts)
- 256-bit key = 2^256 possible keys (extremely secure)
- GCM provides authentication (prevents tampering)
- Industry standard for banking applications

**Key Components**:
1. **Key**: 256-bit secret (must be kept secure)
2. **IV (Initialization Vector)**: Random value per encryption
3. **Auth Tag**: Verifies data wasn't tampered with
4. **Plaintext**: Original sensitive data
5. **Ciphertext**: Encrypted output

### RSA (Rivest-Shamir-Adleman)

**RSA-2048/4096**:
- Asymmetric encryption (public/private key pair)
- Used for key exchange, digital signatures
- Slower than AES, used for small data
- Public key can be shared openly

**Use Cases**:
1. **Key Exchange**: Encrypt AES key with RSA public key
2. **Digital Signatures**: Prove message authenticity
3. **Certificate Authentication**: SSL/TLS certificates

### HTTPS/TLS (Transport Layer Security)

**TLS 1.3** (Latest):
- Encrypts data in transit
- Prevents man-in-the-middle attacks
- Requires SSL/TLS certificate
- Mandatory for banking APIs

**TLS Handshake**:
```
Client                          Server
  |                               |
  |-- ClientHello --------------> |
  |                               |
  | <-- ServerHello + Certificate-|
  |                               |
  |-- Key Exchange -------------> |
  |                               |
  | <-- Finished (Encrypted) ---- |
  |                               |
  |-- Data (Encrypted) ---------> |
```

### Data Classification for Banking

**Level 1 - Highly Sensitive** (Encrypt Always):
- Credit/Debit card numbers (PAN)
- CVV/CVC codes
- Social Security Numbers (SSN)
- Account passwords/PINs
- Biometric data

**Level 2 - Sensitive** (Encrypt at Rest):
- Account numbers
- Customer names + full address
- Date of birth
- Transaction history
- Income information

**Level 3 - Internal** (Hash/Tokenize):
- Email addresses
- Phone numbers
- User IDs

**Level 4 - Public** (No encryption needed):
- Bank branch locations
- Public interest rates
- General marketing content

---

## 💻 Example 1: AES-256-GCM Encryption for Banking Data

**Scenario**: Encrypt customer sensitive data (SSN, account numbers, credit cards) before storing in database.

### Implementation

```javascript
const crypto = require('crypto');

/**
 * AES-256-GCM Encryption Service for Banking Data
 * Handles encryption/decryption of sensitive customer information
 */
class BankingEncryptionService {
  constructor() {
    // In production, load from secure key management service (AWS KMS, HashiCorp Vault)
    this.masterKey = process.env.ENCRYPTION_MASTER_KEY;
    
    if (!this.masterKey || this.masterKey.length !== 64) {
      throw new Error('ENCRYPTION_MASTER_KEY must be 64 hex characters (256 bits)');
    }
    
    // Convert hex string to Buffer
    this.keyBuffer = Buffer.from(this.masterKey, 'hex');
    
    // Encryption algorithm
    this.algorithm = 'aes-256-gcm';
    this.ivLength = 16; // 128 bits for GCM
    this.authTagLength = 16; // 128 bits
  }
  
  /**
   * Encrypt sensitive data
   * @param {string} plaintext - Data to encrypt
   * @returns {object} - Encrypted data with IV and auth tag
   */
  encrypt(plaintext) {
    if (!plaintext) {
      throw new Error('Plaintext cannot be empty');
    }
    
    // Generate random IV (must be unique per encryption)
    const iv = crypto.randomBytes(this.ivLength);
    
    // Create cipher with key and IV
    const cipher = crypto.createCipheriv(this.algorithm, this.keyBuffer, iv);
    
    // Encrypt data
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    // Get authentication tag (prevents tampering)
    const authTag = cipher.getAuthTag();
    
    // Return all components needed for decryption
    return {
      ciphertext: encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
      algorithm: this.algorithm
    };
  }
  
  /**
   * Decrypt encrypted data
   * @param {object} encryptedData - Object with ciphertext, iv, authTag
   * @returns {string} - Decrypted plaintext
   */
  decrypt(encryptedData) {
    const { ciphertext, iv, authTag } = encryptedData;
    
    if (!ciphertext || !iv || !authTag) {
      throw new Error('Missing required encryption components');
    }
    
    // Convert hex strings back to Buffers
    const ivBuffer = Buffer.from(iv, 'hex');
    const authTagBuffer = Buffer.from(authTag, 'hex');
    
    // Create decipher
    const decipher = crypto.createDecipheriv(this.algorithm, this.keyBuffer, ivBuffer);
    
    // Set auth tag for verification
    decipher.setAuthTag(authTagBuffer);
    
    // Decrypt data
    let decrypted = decipher.update(ciphertext, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  /**
   * Encrypt object fields selectively
   * @param {object} data - Object with sensitive fields
   * @param {array} fieldsToEncrypt - Array of field names to encrypt
   * @returns {object} - Object with encrypted fields
   */
  encryptFields(data, fieldsToEncrypt) {
    const result = { ...data };
    
    for (const field of fieldsToEncrypt) {
      if (data[field]) {
        result[field] = this.encrypt(String(data[field]));
      }
    }
    
    return result;
  }
  
  /**
   * Decrypt object fields selectively
   * @param {object} data - Object with encrypted fields
   * @param {array} fieldsToDecrypt - Array of field names to decrypt
   * @returns {object} - Object with decrypted fields
   */
  decryptFields(data, fieldsToDecrypt) {
    const result = { ...data };
    
    for (const field of fieldsToDecrypt) {
      if (data[field] && typeof data[field] === 'object') {
        try {
          result[field] = this.decrypt(data[field]);
        } catch (error) {
          console.error(`Failed to decrypt field ${field}:`, error.message);
          result[field] = null;
        }
      }
    }
    
    return result;
  }
}

/**
 * Customer Data Model with Encryption
 */
class SecureCustomerModel {
  constructor(encryptionService) {
    this.encryption = encryptionService;
    this.customers = new Map(); // In-memory store (use database in production)
  }
  
  /**
   * Create customer with encrypted sensitive data
   */
  async createCustomer(customerData) {
    const {
      firstName,
      lastName,
      email,
      ssn,
      accountNumber,
      dateOfBirth,
      address
    } = customerData;
    
    const customerId = `CUST-${Date.now()}`;
    
    // Encrypt highly sensitive fields
    const encryptedCustomer = {
      id: customerId,
      firstName, // Not encrypted (needed for queries)
      lastName,  // Not encrypted
      email,     // Not encrypted (needed for login)
      
      // Encrypted fields
      ssn: this.encryption.encrypt(ssn),
      accountNumber: this.encryption.encrypt(accountNumber),
      dateOfBirth: this.encryption.encrypt(dateOfBirth),
      
      // Partially encrypted address (encrypt street only)
      address: {
        street: this.encryption.encrypt(address.street),
        city: address.city,    // Not encrypted (for analytics)
        state: address.state,  // Not encrypted
        zipCode: address.zipCode
      },
      
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    this.customers.set(customerId, encryptedCustomer);
    
    return {
      customerId,
      message: 'Customer created with encrypted sensitive data'
    };
  }
  
  /**
   * Get customer with decrypted data
   */
  async getCustomer(customerId) {
    const encryptedCustomer = this.customers.get(customerId);
    
    if (!encryptedCustomer) {
      throw new Error('Customer not found');
    }
    
    // Decrypt sensitive fields
    return {
      ...encryptedCustomer,
      ssn: this.encryption.decrypt(encryptedCustomer.ssn),
      accountNumber: this.encryption.decrypt(encryptedCustomer.accountNumber),
      dateOfBirth: this.encryption.decrypt(encryptedCustomer.dateOfBirth),
      address: {
        ...encryptedCustomer.address,
        street: this.encryption.decrypt(encryptedCustomer.address.street)
      }
    };
  }
  
  /**
   * Get customer with masked sensitive data (for non-privileged access)
   */
  async getCustomerMasked(customerId) {
    const customer = await this.getCustomer(customerId);
    
    return {
      id: customer.id,
      firstName: customer.firstName,
      lastName: customer.lastName,
      email: customer.email,
      ssn: this.maskSSN(customer.ssn),
      accountNumber: this.maskAccountNumber(customer.accountNumber),
      dateOfBirth: customer.dateOfBirth.substring(0, 4) + '-XX-XX', // Show year only
      address: {
        street: '*** Encrypted ***',
        city: customer.address.city,
        state: customer.address.state,
        zipCode: customer.address.zipCode
      }
    };
  }
  
  maskSSN(ssn) {
    // XXX-XX-1234 (show last 4 digits only)
    return `XXX-XX-${ssn.slice(-4)}`;
  }
  
  maskAccountNumber(accountNumber) {
    // ****5678 (show last 4 digits)
    return `****${accountNumber.slice(-4)}`;
  }
}

// Usage Example
console.log('=== AES-256-GCM Banking Data Encryption ===\n');

// Initialize encryption service
const encryptionService = new BankingEncryptionService();
const customerModel = new SecureCustomerModel(encryptionService);

// Test 1: Encrypt/Decrypt simple string
console.log('1. Basic Encryption Test:');
const sensitiveData = '123-45-6789'; // SSN
const encrypted = encryptionService.encrypt(sensitiveData);
console.log('Plaintext:', sensitiveData);
console.log('Encrypted:', {
  ciphertext: encrypted.ciphertext.substring(0, 40) + '...',
  iv: encrypted.iv,
  authTag: encrypted.authTag
});

const decrypted = encryptionService.decrypt(encrypted);
console.log('Decrypted:', decrypted);
console.log('Match:', sensitiveData === decrypted ? '✅' : '❌');

// Test 2: Create customer with encrypted data
console.log('\n2. Creating Customer with Encrypted Data:');
customerModel.createCustomer({
  firstName: 'John',
  lastName: 'Doe',
  email: 'john.doe@example.com',
  ssn: '123-45-6789',
  accountNumber: '9876543210',
  dateOfBirth: '1985-06-15',
  address: {
    street: '123 Main Street, Apt 4B',
    city: 'New York',
    state: 'NY',
    zipCode: '10001'
  }
}).then(result => {
  console.log('Customer created:', result.customerId);
  
  // Test 3: Retrieve with full decryption (privileged access)
  console.log('\n3. Retrieve Customer (Full Access):');
  return customerModel.getCustomer(result.customerId);
}).then(customer => {
  console.log('Customer ID:', customer.id);
  console.log('Name:', customer.firstName, customer.lastName);
  console.log('SSN:', customer.ssn);
  console.log('Account:', customer.accountNumber);
  console.log('DOB:', customer.dateOfBirth);
  console.log('Street:', customer.address.street);
  
  // Test 4: Retrieve with masking (limited access)
  console.log('\n4. Retrieve Customer (Masked - Limited Access):');
  return customerModel.getCustomerMasked(customer.id);
}).then(maskedCustomer => {
  console.log('Customer ID:', maskedCustomer.id);
  console.log('Name:', maskedCustomer.firstName, maskedCustomer.lastName);
  console.log('SSN:', maskedCustomer.ssn);
  console.log('Account:', maskedCustomer.accountNumber);
  console.log('DOB:', maskedCustomer.dateOfBirth);
  console.log('Street:', maskedCustomer.address.street);
});

// Test 5: Encrypt multiple fields at once
console.log('\n5. Bulk Field Encryption:');
const customerData = {
  name: 'Jane Smith',
  ssn: '987-65-4321',
  creditCard: '4532-1234-5678-9010',
  salary: '85000'
};

const encryptedData = encryptionService.encryptFields(customerData, ['ssn', 'creditCard', 'salary']);
console.log('Original:', customerData);
console.log('Encrypted SSN:', encryptedData.ssn.ciphertext.substring(0, 30) + '...');
console.log('Encrypted CC:', encryptedData.creditCard.ciphertext.substring(0, 30) + '...');

const decryptedData = encryptionService.decryptFields(encryptedData, ['ssn', 'creditCard', 'salary']);
console.log('Decrypted:', decryptedData);
```

### Running the Example

```bash
# Set encryption key (256-bit = 64 hex characters)
export ENCRYPTION_MASTER_KEY="0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"

# Run the script
node aes_encryption_example.js
```

### Expected Output

```
=== AES-256-GCM Banking Data Encryption ===

1. Basic Encryption Test:
Plaintext: 123-45-6789
Encrypted: {
  ciphertext: 'a3f7b2c9d4e5f6a7b8c9d0e1f2a3b4c5...',
  iv: '9f8e7d6c5b4a39281706f5e4d3c2b1a0',
  authTag: 'e4d3c2b1a09f8e7d6c5b4a3928170615'
}
Decrypted: 123-45-6789
Match: ✅

2. Creating Customer with Encrypted Data:
Customer created: CUST-1700000000000

3. Retrieve Customer (Full Access):
Customer ID: CUST-1700000000000
Name: John Doe
SSN: 123-45-6789
Account: 9876543210
DOB: 1985-06-15
Street: 123 Main Street, Apt 4B

4. Retrieve Customer (Masked - Limited Access):
Customer ID: CUST-1700000000000
Name: John Doe
SSN: XXX-XX-6789
Account: ****3210
DOB: 1985-XX-XX
Street: *** Encrypted ***

5. Bulk Field Encryption:
Original: { name: 'Jane Smith', ssn: '987-65-4321', ... }
Encrypted SSN: b5c6d7e8f9a0b1c2d3e4f5a6b7c8...
Decrypted: { name: 'Jane Smith', ssn: '987-65-4321', ... }
```

### Key Takeaways

1. **AES-256-GCM** provides both encryption and authentication
2. **IV must be unique** for each encryption (never reuse)
3. **Store IV and auth tag** alongside ciphertext
4. **Master key management** is critical (use KMS in production)
5. **Selective encryption** - only encrypt sensitive fields
6. **Data masking** for different access levels
7. **Never log** plaintext sensitive data
8. **Encrypt before storing** in database

---

## 💻 Example 2: RSA Asymmetric Encryption for Key Exchange

**Scenario**: Use RSA to securely exchange AES encryption keys between client and server.

### Implementation

```javascript
const crypto = require('crypto');
const fs = require('fs');

/**
 * RSA Key Pair Management for Banking Systems
 * Handles secure key exchange and digital signatures
 */
class RSAKeyManager {
  constructor() {
    this.publicKey = null;
    this.privateKey = null;
  }
  
  /**
   * Generate RSA-2048 key pair
   * Production: Use RSA-4096 for higher security
   */
  generateKeyPair() {
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048, // 2048 bits
      publicKeyEncoding: {
        type: 'spki',
        format: 'pem'
      },
      privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem',
        cipher: 'aes-256-cbc',
        passphrase: process.env.PRIVATE_KEY_PASSPHRASE || 'secure-passphrase'
      }
    });
    
    this.publicKey = publicKey;
    this.privateKey = privateKey;
    
    return { publicKey, privateKey };
  }
  
  /**
   * Load existing keys from files
   */
  loadKeys(publicKeyPath, privateKeyPath, passphrase) {
    this.publicKey = fs.readFileSync(publicKeyPath, 'utf8');
    this.privateKey = fs.readFileSync(privateKeyPath, 'utf8');
    this.passphrase = passphrase;
  }
  
  /**
   * Save keys to files
   */
  saveKeys(publicKeyPath, privateKeyPath) {
    fs.writeFileSync(publicKeyPath, this.publicKey);
    fs.writeFileSync(privateKeyPath, this.privateKey);
    console.log(`Keys saved to ${publicKeyPath} and ${privateKeyPath}`);
  }
  
  /**
   * Encrypt data with public key
   * Used by client to encrypt AES key for server
   */
  encryptWithPublicKey(data) {
    const buffer = Buffer.from(data, 'utf8');
    
    const encrypted = crypto.publicEncrypt(
      {
        key: this.publicKey,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      buffer
    );
    
    return encrypted.toString('base64');
  }
  
  /**
   * Decrypt data with private key
   * Used by server to decrypt AES key from client
   */
  decryptWithPrivateKey(encryptedData) {
    const buffer = Buffer.from(encryptedData, 'base64');
    
    const decrypted = crypto.privateDecrypt(
      {
        key: this.privateKey,
        passphrase: this.passphrase || process.env.PRIVATE_KEY_PASSPHRASE,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      buffer
    );
    
    return decrypted.toString('utf8');
  }
  
  /**
   * Sign data with private key (digital signature)
   * Proves data came from holder of private key
   */
  sign(data) {
    const sign = crypto.createSign('SHA256');
    sign.update(data);
    sign.end();
    
    const signature = sign.sign(
      {
        key: this.privateKey,
        passphrase: this.passphrase || process.env.PRIVATE_KEY_PASSPHRASE
      },
      'base64'
    );
    
    return signature;
  }
  
  /**
   * Verify signature with public key
   * Confirms data hasn't been tampered with
   */
  verify(data, signature) {
    const verify = crypto.createVerify('SHA256');
    verify.update(data);
    verify.end();
    
    return verify.verify(this.publicKey, signature, 'base64');
  }
}

/**
 * Secure Session Manager using RSA + AES hybrid encryption
 * Client encrypts AES session key with server's RSA public key
 * All subsequent data encrypted with fast AES using session key
 */
class SecureSessionManager {
  constructor() {
    this.rsaManager = new RSAKeyManager();
    this.sessions = new Map();
  }
  
  /**
   * Server: Initialize with RSA keys
   */
  initializeServer() {
    // Generate or load RSA key pair
    this.rsaManager.generateKeyPair();
    console.log('✅ Server RSA keys generated');
    
    // Public key can be shared with clients
    return this.rsaManager.publicKey;
  }
  
  /**
   * Client: Establish secure session
   * 1. Generate random AES session key
   * 2. Encrypt session key with server's RSA public key
   * 3. Send encrypted session key to server
   */
  establishSession(serverPublicKey) {
    // Generate random 256-bit AES session key
    const sessionKey = crypto.randomBytes(32).toString('hex');
    const sessionId = `SESSION-${Date.now()}-${Math.random().toString(36).substring(7)}`;
    
    // Encrypt session key with server's public key
    const tempRSA = new RSAKeyManager();
    tempRSA.publicKey = serverPublicKey;
    const encryptedSessionKey = tempRSA.encryptWithPublicKey(sessionKey);
    
    // Store session locally (client-side)
    this.sessions.set(sessionId, {
      sessionKey,
      createdAt: new Date()
    });
    
    console.log(`✅ Client: Session ${sessionId} established`);
    console.log(`   Session key (first 16 chars): ${sessionKey.substring(0, 16)}...`);
    console.log(`   Encrypted key length: ${encryptedSessionKey.length} bytes`);
    
    return {
      sessionId,
      encryptedSessionKey // Send this to server
    };
  }
  
  /**
   * Server: Accept session request
   * 1. Decrypt session key with private RSA key
   * 2. Store session for future use
   */
  acceptSession(sessionId, encryptedSessionKey) {
    // Decrypt session key with server's private key
    const sessionKey = this.rsaManager.decryptWithPrivateKey(encryptedSessionKey);
    
    // Store session server-side
    this.sessions.set(sessionId, {
      sessionKey,
      createdAt: new Date(),
      lastUsed: new Date()
    });
    
    console.log(`✅ Server: Session ${sessionId} accepted`);
    console.log(`   Decrypted session key (first 16 chars): ${sessionKey.substring(0, 16)}...`);
    
    return { success: true, sessionId };
  }
  
  /**
   * Encrypt data using session AES key
   */
  encryptWithSession(sessionId, data) {
    const session = this.sessions.get(sessionId);
    if (!session) {
      throw new Error('Invalid session');
    }
    
    const keyBuffer = Buffer.from(session.sessionKey, 'hex');
    const iv = crypto.randomBytes(16);
    
    const cipher = crypto.createCipheriv('aes-256-gcm', keyBuffer, iv);
    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      ciphertext: encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  }
  
  /**
   * Decrypt data using session AES key
   */
  decryptWithSession(sessionId, encryptedData) {
    const session = this.sessions.get(sessionId);
    if (!session) {
      throw new Error('Invalid session');
    }
    
    const { ciphertext, iv, authTag } = encryptedData;
    const keyBuffer = Buffer.from(session.sessionKey, 'hex');
    const ivBuffer = Buffer.from(iv, 'hex');
    const authTagBuffer = Buffer.from(authTag, 'hex');
    
    const decipher = crypto.createDecipheriv('aes-256-gcm', keyBuffer, ivBuffer);
    decipher.setAuthTag(authTagBuffer);
    
    let decrypted = decipher.update(ciphertext, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

/**
 * Banking Transaction with End-to-End Encryption
 */
class SecureBankingAPI {
  constructor() {
    this.sessionManager = new SecureSessionManager();
  }
  
  /**
   * Server initialization
   */
  startServer() {
    const publicKey = this.sessionManager.initializeServer();
    return publicKey;
  }
  
  /**
   * Client: Send encrypted transaction
   */
  sendTransaction(sessionId, transaction) {
    const transactionJSON = JSON.stringify(transaction);
    
    // Encrypt with AES session key
    const encrypted = this.sessionManager.encryptWithSession(sessionId, transactionJSON);
    
    console.log('\n📤 Client: Sending encrypted transaction');
    console.log('   Transaction size:', transactionJSON.length, 'bytes');
    console.log('   Encrypted size:', encrypted.ciphertext.length / 2, 'bytes');
    
    return encrypted;
  }
  
  /**
   * Server: Receive and decrypt transaction
   */
  receiveTransaction(sessionId, encryptedTransaction) {
    // Decrypt with AES session key
    const decrypted = this.sessionManager.decryptWithSession(sessionId, encryptedTransaction);
    const transaction = JSON.parse(decrypted);
    
    console.log('\n📥 Server: Received encrypted transaction');
    console.log('   Decrypted transaction:', transaction);
    
    return transaction;
  }
}

// Usage Example
console.log('=== RSA + AES Hybrid Encryption ===\n');

// Simulate server and client
const server = new SecureBankingAPI();
const client = new SecureBankingAPI();

// Step 1: Server generates RSA keys and shares public key
console.log('Step 1: Server Initialization');
const serverPublicKey = server.startServer();

// Step 2: Client establishes secure session
console.log('\nStep 2: Client Establishes Session');
const { sessionId, encryptedSessionKey } = client.sessionManager.establishSession(serverPublicKey);

// Step 3: Server accepts session
console.log('\nStep 3: Server Accepts Session');
server.sessionManager.acceptSession(sessionId, encryptedSessionKey);

// Step 4: Client sends encrypted transaction
const transaction = {
  from: 'ACC-123456',
  to: 'ACC-789012',
  amount: 5000,
  currency: 'USD',
  description: 'Wire transfer',
  timestamp: new Date().toISOString()
};

const encryptedTxn = client.sendTransaction(sessionId, transaction);

// Step 5: Server receives and decrypts
const decryptedTxn = server.receiveTransaction(sessionId, encryptedTxn);

// Verify transaction integrity
console.log('\n✅ Transaction verified:');
console.log('   From:', decryptedTxn.from);
console.log('   To:', decryptedTxn.to);
console.log('   Amount:', decryptedTxn.amount, decryptedTxn.currency);

// Test 2: Digital Signatures
console.log('\n\n=== Digital Signatures ===\n');

const rsaManager = new RSAKeyManager();
rsaManager.generateKeyPair();

const message = 'Transfer $10000 from ACC-123 to ACC-456';
console.log('Original message:', message);

// Sign message
const signature = rsaManager.sign(message);
console.log('Signature:', signature.substring(0, 40) + '...');

// Verify signature
const isValid = rsaManager.verify(message, signature);
console.log('Signature valid:', isValid ? '✅' : '❌');

// Test tampering
const tamperedMessage = 'Transfer $99999 from ACC-123 to ACC-456';
const isTamperedValid = rsaManager.verify(tamperedMessage, signature);
console.log('Tampered message valid:', isTamperedValid ? '✅' : '❌');
```

### Running the Example

```bash
# Run RSA encryption example
node rsa_encryption_example.js
```

### Expected Output

```
=== RSA + AES Hybrid Encryption ===

Step 1: Server Initialization
✅ Server RSA keys generated

Step 2: Client Establishes Session
✅ Client: Session SESSION-1700000000-abc123 established
   Session key (first 16 chars): a3f7b2c9d4e5f6a7...
   Encrypted key length: 344 bytes

Step 3: Server Accepts Session
✅ Server: Session SESSION-1700000000-abc123 accepted
   Decrypted session key (first 16 chars): a3f7b2c9d4e5f6a7...

📤 Client: Sending encrypted transaction
   Transaction size: 156 bytes
   Encrypted size: 172 bytes

📥 Server: Received encrypted transaction
   Decrypted transaction: {
     from: 'ACC-123456',
     to: 'ACC-789012',
     amount: 5000,
     currency: 'USD',
     description: 'Wire transfer'
   }

✅ Transaction verified:
   From: ACC-123456
   To: ACC-789012
   Amount: 5000 USD

=== Digital Signatures ===

Original message: Transfer $10000 from ACC-123 to ACC-456
Signature: kF8mD3xQ9vL2jN7pW1cR5tY8uH4bG6zA...
Signature valid: ✅
Tampered message valid: ❌
```

### Key Takeaways

1. **RSA for key exchange** - Encrypt AES key with public key
2. **AES for data** - Fast symmetric encryption for bulk data
3. **Hybrid approach** - Best of both worlds (security + performance)
4. **Digital signatures** - Verify message authenticity
5. **Key size matters** - RSA-2048 minimum, RSA-4096 for banking
6. **Never expose private keys** - Keep them secure
7. **Session keys** - Rotate frequently for security

---

## 💻 Example 3: HTTPS/TLS Setup for Secure Banking API

**Scenario**: Configure HTTPS with TLS 1.3 for secure API communications in banking application.

### Implementation

```javascript
const https = require('https');
const http = require('http');
const fs = require('fs');
const express = require('express');
const helmet = require('helmet');
const crypto = require('crypto');

/**
 * Secure Banking API Server with HTTPS/TLS
 * Implements best practices for secure communication
 */
class SecureBankingServer {
  constructor() {
    this.app = express();
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  /**
   * Setup security middleware
   */
  setupMiddleware() {
    // Helmet: Security headers
    this.app.use(helmet({
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'"],
          scriptSrc: ["'self'"],
          imgSrc: ["'self'", 'data:', 'https:'],
        },
      },
      hsts: {
        maxAge: 31536000, // 1 year
        includeSubDomains: true,
        preload: true
      },
      referrerPolicy: {
        policy: 'strict-origin-when-cross-origin'
      }
    }));
    
    // Parse JSON bodies
    this.app.use(express.json({ limit: '10mb' }));
    
    // Request logging
    this.app.use((req, res, next) => {
      console.log(`${new Date().toISOString()} - ${req.method} ${req.path}`);
      next();
    });
    
    // Force HTTPS redirect (if accessed via HTTP)
    this.app.use((req, res, next) => {
      if (!req.secure && req.get('x-forwarded-proto') !== 'https') {
        return res.redirect('https://' + req.get('host') + req.url);
      }
      next();
    });
  }
  
  /**
   * Setup API routes
   */
  setupRoutes() {
    // Health check
    this.app.get('/health', (req, res) => {
      res.json({
        status: 'healthy',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        tls: req.secure ? 'enabled' : 'disabled'
      });
    });
    
    // Get account balance (requires authentication)
    this.app.get('/api/accounts/:accountId/balance', this.requireAuth, (req, res) => {
      const { accountId } = req.params;
      
      // Simulate database lookup
      const balance = Math.floor(Math.random() * 100000);
      
      res.json({
        accountId,
        balance,
        currency: 'USD',
        timestamp: new Date().toISOString()
      });
    });
    
    // Transfer funds (requires authentication)
    this.app.post('/api/transfers', this.requireAuth, (req, res) => {
      const { from, to, amount, description } = req.body;
      
      // Validate input
      if (!from || !to || !amount) {
        return res.status(400).json({
          error: 'Missing required fields'
        });
      }
      
      if (amount <= 0 || amount > 100000) {
        return res.status(400).json({
          error: 'Invalid amount'
        });
      }
      
      // Process transfer (simulated)
      const transferId = `TXN-${Date.now()}`;
      
      res.json({
        transferId,
        from,
        to,
        amount,
        description,
        status: 'completed',
        timestamp: new Date().toISOString()
      });
    });
    
    // Certificate info endpoint
    this.app.get('/api/certificate-info', (req, res) => {
      if (!req.socket.getPeerCertificate) {
        return res.json({ tls: false });
      }
      
      const cert = req.socket.getPeerCertificate();
      
      res.json({
        tls: true,
        protocol: req.socket.getProtocol?.() || 'Unknown',
        cipher: req.socket.getCipher?.() || 'Unknown',
        certificate: {
          subject: cert.subject,
          issuer: cert.issuer,
          validFrom: cert.valid_from,
          validTo: cert.valid_to
        }
      });
    });
    
    // 404 handler
    this.app.use((req, res) => {
      res.status(404).json({
        error: 'Not found',
        path: req.path
      });
    });
    
    // Error handler
    this.app.use((err, req, res, next) => {
      console.error('Error:', err);
      res.status(500).json({
        error: 'Internal server error',
        message: process.env.NODE_ENV === 'development' ? err.message : undefined
      });
    });
  }
  
  /**
   * Authentication middleware (simplified for demo)
   */
  requireAuth(req, res, next) {
    const apiKey = req.headers['x-api-key'];
    
    if (!apiKey || apiKey !== 'demo-api-key-12345') {
      return res.status(401).json({
        error: 'Unauthorized',
        message: 'Valid API key required'
      });
    }
    
    next();
  }
  
  /**
   * Generate self-signed certificate for development
   * Production: Use Let's Encrypt or commercial CA
   */
  generateSelfSignedCert() {
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: {
        type: 'spki',
        format: 'pem'
      },
      privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem'
      }
    });
    
    // In production, get proper certificate from CA
    // For development, create self-signed cert files
    const certDir = './certs';
    if (!fs.existsSync(certDir)) {
      fs.mkdirSync(certDir);
    }
    
    fs.writeFileSync(`${certDir}/private-key.pem`, privateKey);
    fs.writeFileSync(`${certDir}/public-key.pem`, publicKey);
    
    console.log('✅ Self-signed certificate generated in ./certs/');
    console.log('⚠️  For production, use proper CA-signed certificate');
  }
  
  /**
   * Start HTTPS server
   */
  startHTTPS(port = 3443) {
    // Load TLS certificate and private key
    let tlsOptions;
    
    try {
      tlsOptions = {
        key: fs.readFileSync('./certs/private-key.pem'),
        cert: fs.readFileSync('./certs/certificate.pem'),
        
        // TLS configuration
        minVersion: 'TLSv1.3', // Require TLS 1.3
        maxVersion: 'TLSv1.3',
        
        // Cipher suites (strongest first)
        ciphers: [
          'TLS_AES_256_GCM_SHA384',
          'TLS_CHACHA20_POLY1305_SHA256',
          'TLS_AES_128_GCM_SHA256'
        ].join(':'),
        
        // Enable HSTS
        honorCipherOrder: true
      };
    } catch (error) {
      console.error('❌ Certificate files not found');
      console.log('Generating self-signed certificate...');
      this.generateSelfSignedCert();
      
      // For demo, use existing keys
      tlsOptions = {
        key: fs.readFileSync('./certs/private-key.pem'),
        cert: fs.readFileSync('./certs/public-key.pem')
      };
    }
    
    // Create HTTPS server
    const server = https.createServer(tlsOptions, this.app);
    
    server.listen(port, () => {
      console.log(`\n🔒 Secure Banking API Server`);
      console.log(`✅ HTTPS server running on https://localhost:${port}`);
      console.log(`📋 TLS Version: 1.3`);
      console.log(`🔐 Strong cipher suites enabled`);
      console.log(`\nAvailable endpoints:`);
      console.log(`  GET  /health`);
      console.log(`  GET  /api/accounts/:id/balance`);
      console.log(`  POST /api/transfers`);
      console.log(`  GET  /api/certificate-info`);
      console.log(`\nTest with: curl -k -H "x-api-key: demo-api-key-12345" https://localhost:${port}/health`);
    });
    
    return server;
  }
  
  /**
   * Start HTTP server (for redirect only)
   */
  startHTTP(port = 3080) {
    const httpServer = http.createServer((req, res) => {
      res.writeHead(301, {
        'Location': `https://${req.headers.host.replace(/:\d+/, ':3443')}${req.url}`
      });
      res.end();
    });
    
    httpServer.listen(port, () => {
      console.log(`↪️  HTTP redirect server on http://localhost:${port} (redirects to HTTPS)`);
    });
    
    return httpServer;
  }
}

// Usage Example
if (require.main === module) {
  const server = new SecureBankingServer();
  
  // Start HTTPS server
  server.startHTTPS(3443);
  
  // Start HTTP redirect server
  server.startHTTP(3080);
  
  // Handle graceful shutdown
  process.on('SIGTERM', () => {
    console.log('\n🛑 SIGTERM received, shutting down gracefully...');
    process.exit(0);
  });
}

module.exports = SecureBankingServer;
```

### Certificate Generation Script

```bash
#!/bin/bash
# generate-cert.sh - Generate self-signed certificate for development

mkdir -p certs

# Generate private key
openssl genrsa -out certs/private-key.pem 2048

# Generate certificate signing request (CSR)
openssl req -new -key certs/private-key.pem -out certs/csr.pem \
  -subj "/C=US/ST=NY/L=New York/O=SecureBank/OU=IT/CN=localhost"

# Generate self-signed certificate (valid for 365 days)
openssl x509 -req -days 365 -in certs/csr.pem \
  -signkey certs/private-key.pem -out certs/certificate.pem

echo "✅ Certificate generated in ./certs/"
echo "⚠️  This is a self-signed certificate for development only"
echo "⚠️  For production, use Let's Encrypt or commercial CA"

# Cleanup
rm certs/csr.pem
```

### Client Test Script

```javascript
// test-client.js - Test HTTPS API

const https = require('https');

// Allow self-signed certificates (development only)
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';

function makeRequest(endpoint, method = 'GET', data = null) {
  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'localhost',
      port: 3443,
      path: endpoint,
      method,
      headers: {
        'x-api-key': 'demo-api-key-12345',
        'Content-Type': 'application/json'
      }
    };
    
    const req = https.request(options, (res) => {
      let body = '';
      
      res.on('data', (chunk) => {
        body += chunk;
      });
      
      res.on('end', () => {
        console.log(`\n${method} ${endpoint}`);
        console.log(`Status: ${res.statusCode}`);
        console.log(`Response:`, JSON.parse(body));
        resolve(JSON.parse(body));
      });
    });
    
    req.on('error', reject);
    
    if (data) {
      req.write(JSON.stringify(data));
    }
    
    req.end();
  });
}

async function runTests() {
  console.log('=== Testing Secure Banking API ===');
  
  // Test 1: Health check
  await makeRequest('/health');
  
  // Test 2: Get account balance
  await makeRequest('/api/accounts/ACC-123456/balance');
  
  // Test 3: Transfer funds
  await makeRequest('/api/transfers', 'POST', {
    from: 'ACC-123456',
    to: 'ACC-789012',
    amount: 5000,
    description: 'Rent payment'
  });
  
  // Test 4: Certificate info
  await makeRequest('/api/certificate-info');
}

runTests().catch(console.error);
```

### Running the Example

```bash
# 1. Install dependencies
npm install express helmet

# 2. Generate certificate
chmod +x generate-cert.sh
./generate-cert.sh

# 3. Start server
node secure_server.js

# 4. Test with curl
curl -k -H "x-api-key: demo-api-key-12345" https://localhost:3443/health

# 5. Or run test client
node test-client.js
```

### Expected Output

```
🔒 Secure Banking API Server
✅ HTTPS server running on https://localhost:3443
📋 TLS Version: 1.3
🔐 Strong cipher suites enabled

Available endpoints:
  GET  /health
  GET  /api/accounts/:id/balance
  POST /api/transfers
  GET  /api/certificate-info

↪️  HTTP redirect server on http://localhost:3080 (redirects to HTTPS)

Test with: curl -k -H "x-api-key: demo-api-key-12345" https://localhost:3443/health
```

### Key Takeaways

1. **TLS 1.3** - Latest and most secure protocol
2. **Strong cipher suites** - AES-256-GCM, ChaCha20-Poly1305
3. **HTTPS everywhere** - Force redirect from HTTP
4. **Security headers** - Helmet middleware adds protection
5. **Certificate management** - Proper CA certificates in production
6. **Perfect forward secrecy** - Each session has unique keys
7. **HSTS** - Force HTTPS for future requests

---

## 🎓 Key Takeaways

### Data Encryption Best Practices

1. **Use AES-256-GCM** for data at rest
   - Provides both encryption and authentication
   - Industry standard for banking applications
   - Never reuse IV (initialization vector)

2. **Use RSA-2048/4096** for key exchange
   - Hybrid approach: RSA for keys, AES for data
   - Digital signatures prove authenticity
   - Keep private keys secure (HSM in production)

3. **Implement TLS 1.3** for data in transit
   - Force HTTPS for all banking APIs
   - Use strong cipher suites only
   - Enable HSTS to prevent downgrade attacks

4. **Key Management is Critical**
   - Never hardcode keys in source code
   - Use environment variables or KMS (AWS KMS, Vault)
   - Rotate keys regularly (quarterly minimum)
   - Separate keys per environment (dev/staging/prod)

5. **Encrypt Selectively**
   - Not all data needs encryption (analyze and classify)
   - Encrypt: SSN, credit cards, account numbers, PINs
   - Hash: Passwords (use bcrypt, not encryption)
   - Tokenize: Payment card data (PCI-DSS requirement)

6. **Access Control**
   - Implement role-based access (RBAC)
   - Data masking for limited privileges
   - Audit all access to encrypted data
   - Least privilege principle

7. **Compliance Requirements**
   - **PCI-DSS**: Credit card data must be encrypted
   - **GDPR**: Personal data protection requirements
   - **SOX**: Financial data integrity and security
   - **GLBA**: Banking customer data protection

8. **Performance Considerations**
   - AES is fast (~1GB/s), RSA is slow (~100KB/s)
   - Cache decrypted data carefully (memory leaks)
   - Use hardware acceleration (AES-NI)
   - Encrypt at application layer, not database layer

9. **Security in Depth**
   - Encryption is one layer of many
   - Also implement: authentication, authorization, logging
   - Network segmentation and firewalls
   - Intrusion detection systems

10. **Testing and Monitoring**
    - Test encryption/decryption pipelines
    - Monitor for encryption failures
    - Alert on unusual access patterns
    - Regular security audits

---

## 📚 Additional Resources

- **NIST Guidelines**: [SP 800-175B](https://csrc.nist.gov/publications/detail/sp/800-175b/final)
- **OWASP Cryptographic Storage**: [Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- **PCI-DSS Requirements**: [Official Standards](https://www.pcisecuritystandards.org/)
- **Node.js Crypto**: [Documentation](https://nodejs.org/api/crypto.html)

---

**Next**: Q29 - Database Connection Pooling
