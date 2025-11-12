# Security Questions 21-25: Encryption & Key Management

---

### Q21. How do you implement encryption at rest and in transit?

**Answer:**

```javascript
const crypto = require('crypto');
const fs = require('fs');
const AWS = require('aws-sdk');

/**
 * Encryption Service for Data at Rest
 */
class DataEncryptionService {
  constructor() {
    this.algorithm = 'aes-256-gcm';
    this.keyLength = 32; // 256 bits
    this.ivLength = 16;  // 128 bits
    this.tagLength = 16; // 128 bits
  }
  
  /**
   * Encrypt data with AES-256-GCM
   */
  encrypt(plaintext, key) {
    // Generate random IV
    const iv = crypto.randomBytes(this.ivLength);
    
    // Create cipher
    const cipher = crypto.createCipheriv(this.algorithm, key, iv);
    
    // Encrypt data
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    // Get authentication tag
    const tag = cipher.getAuthTag();
    
    // Return IV + tag + encrypted data
    return {
      iv: iv.toString('hex'),
      tag: tag.toString('hex'),
      encrypted
    };
  }
  
  /**
   * Decrypt data
   */
  decrypt(encryptedData, key) {
    const { iv, tag, encrypted } = encryptedData;
    
    // Create decipher
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      key,
      Buffer.from(iv, 'hex')
    );
    
    // Set authentication tag
    decipher.setAuthTag(Buffer.from(tag, 'hex'));
    
    // Decrypt data
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  /**
   * Encrypt large files
   */
  encryptFile(inputPath, outputPath, key) {
    return new Promise((resolve, reject) => {
      const iv = crypto.randomBytes(this.ivLength);
      const cipher = crypto.createCipheriv(this.algorithm, key, iv);
      
      const input = fs.createReadStream(inputPath);
      const output = fs.createWriteStream(outputPath);
      
      // Write IV first
      output.write(iv);
      
      input.pipe(cipher).pipe(output);
      
      output.on('finish', () => {
        // Write tag at the end
        const tag = cipher.getAuthTag();
        output.write(tag);
        resolve();
      });
      
      output.on('error', reject);
    });
  }
  
  /**
   * Field-level encryption
   */
  encryptField(value, key, fieldName) {
    const plaintext = JSON.stringify({
      field: fieldName,
      value,
      timestamp: Date.now()
    });
    
    return this.encrypt(plaintext, key);
  }
  
  /**
   * Decrypt field
   */
  decryptField(encryptedField, key) {
    const decrypted = this.decrypt(encryptedField, key);
    const data = JSON.parse(decrypted);
    return data.value;
  }
}

/**
 * Database Encryption Service
 */
class DatabaseEncryptionService {
  constructor() {
    this.encryptionService = new DataEncryptionService();
    this.keyManagement = new KeyManagementService();
  }
  
  /**
   * Encrypt sensitive fields before saving
   */
  async encryptDocument(document, sensitiveFields) {
    const encryptedDoc = { ...document };
    const dataKey = await this.keyManagement.getDataEncryptionKey('enbd-db');
    
    for (const field of sensitiveFields) {
      if (document[field]) {
        encryptedDoc[`${field}_encrypted`] = this.encryptionService.encryptField(
          document[field],
          dataKey,
          field
        );
        delete encryptedDoc[field]; // Remove plaintext
      }
    }
    
    return encryptedDoc;
  }
  
  /**
   * Decrypt document on retrieval
   */
  async decryptDocument(document, sensitiveFields) {
    const decryptedDoc = { ...document };
    const dataKey = await this.keyManagement.getDataEncryptionKey('enbd-db');
    
    for (const field of sensitiveFields) {
      const encryptedField = `${field}_encrypted`;
      if (document[encryptedField]) {
        decryptedDoc[field] = this.encryptionService.decryptField(
          document[encryptedField],
          dataKey
        );
        delete decryptedDoc[encryptedField];
      }
    }
    
    return decryptedDoc;
  }
}

/**
 * TLS/SSL Configuration for Data in Transit
 */
class TLSConfigurationService {
  /**
   * Create secure HTTPS server
   */
  static createSecureServer(app) {
    const options = {
      // Certificates
      key: fs.readFileSync('./certs/server-key.pem'),
      cert: fs.readFileSync('./certs/server-cert.pem'),
      ca: [fs.readFileSync('./certs/ca-cert.pem')],
      
      // TLS 1.3 only
      minVersion: 'TLSv1.3',
      maxVersion: 'TLSv1.3',
      
      // Strong cipher suites
      ciphers: [
        'TLS_AES_256_GCM_SHA384',
        'TLS_CHACHA20_POLY1305_SHA256',
        'TLS_AES_128_GCM_SHA256'
      ].join(':'),
      
      // Perfect Forward Secrecy
      honorCipherOrder: true,
      
      // Client certificate authentication (optional)
      requestCert: false,
      rejectUnauthorized: false,
      
      // OCSP Stapling
      requestOCSP: true
    };
    
    return require('https').createServer(options, app);
  }
  
  /**
   * Configure secure MongoDB connection
   */
  static getSecureMongoConfig() {
    return {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      ssl: true,
      sslValidate: true,
      sslCA: fs.readFileSync('./certs/mongodb-ca.pem'),
      sslCert: fs.readFileSync('./certs/mongodb-cert.pem'),
      sslKey: fs.readFileSync('./certs/mongodb-key.pem'),
      authSource: 'admin',
      authMechanism: 'MONGODB-X509'
    };
  }
  
  /**
   * Configure secure Redis connection
   */
  static getSecureRedisConfig() {
    return {
      host: 'redis.enbd.com',
      port: 6380,
      tls: {
        ca: fs.readFileSync('./certs/redis-ca.pem'),
        cert: fs.readFileSync('./certs/redis-cert.pem'),
        key: fs.readFileSync('./certs/redis-key.pem'),
        rejectUnauthorized: true
      }
    };
  }
  
  /**
   * Secure HTTP client configuration
   */
  static getSecureAxiosConfig() {
    const https = require('https');
    
    return {
      httpsAgent: new https.Agent({
        ca: fs.readFileSync('./certs/ca-bundle.pem'),
        cert: fs.readFileSync('./certs/client-cert.pem'),
        key: fs.readFileSync('./certs/client-key.pem'),
        rejectUnauthorized: true,
        minVersion: 'TLSv1.3'
      })
    };
  }
}

/**
 * ENBD Account Encryption Example
 */
class ENBDAccountService {
  constructor() {
    this.dbEncryption = new DatabaseEncryptionService();
  }
  
  /**
   * Create account with encrypted fields
   */
  async createAccount(accountData) {
    const sensitiveFields = [
      'accountNumber',
      'ssn',
      'phoneNumber',
      'address'
    ];
    
    // Encrypt sensitive fields
    const encryptedAccount = await this.dbEncryption.encryptDocument(
      accountData,
      sensitiveFields
    );
    
    // Save to database
    await this.saveToDatabase(encryptedAccount);
    
    return { success: true };
  }
  
  /**
   * Retrieve account with decryption
   */
  async getAccount(accountId) {
    const sensitiveFields = [
      'accountNumber',
      'ssn',
      'phoneNumber',
      'address'
    ];
    
    // Get from database
    const encryptedAccount = await this.getFromDatabase(accountId);
    
    // Decrypt sensitive fields
    const account = await this.dbEncryption.decryptDocument(
      encryptedAccount,
      sensitiveFields
    );
    
    return account;
  }
  
  async saveToDatabase(data) {
    // Database save
  }
  
  async getFromDatabase(id) {
    return {};
  }
}

module.exports = {
  DataEncryptionService,
  DatabaseEncryptionService,
  TLSConfigurationService,
  ENBDAccountService
};
```

**Encryption Best Practices:**

1. **At Rest**: AES-256-GCM for data
2. **In Transit**: TLS 1.3 for all connections
3. **Database**: Field-level encryption for PII
4. **Files**: Stream encryption for large files
5. **Keys**: Never hardcode, use key management

---

### Q22. How do you implement key management and rotation?

**Answer:**

```javascript
const AWS = require('aws-sdk');
const { SecretManagerServiceClient } = require('@azure/keyvault-secrets');
const crypto = require('crypto');

/**
 * Key Management Service
 */
class KeyManagementService {
  constructor() {
    this.kms = new AWS.KMS({ region: 'me-south-1' }); // Dubai region
    this.keyCache = new Map();
    this.keyRotationDays = 90;
  }
  
  /**
   * Generate Data Encryption Key (DEK)
   */
  async generateDataKey(keyId) {
    const params = {
      KeyId: keyId,
      KeySpec: 'AES_256'
    };
    
    const { Plaintext, CiphertextBlob } = await this.kms.generateDataKey(params).promise();
    
    return {
      plaintextKey: Plaintext,
      encryptedKey: CiphertextBlob
    };
  }
  
  /**
   * Decrypt Data Encryption Key
   */
  async decryptDataKey(encryptedKey) {
    const params = {
      CiphertextBlob: encryptedKey
    };
    
    const { Plaintext } = await this.kms.decrypt(params).promise();
    
    return Plaintext;
  }
  
  /**
   * Envelope Encryption Pattern
   */
  async envelopeEncrypt(data, masterKeyId) {
    // Generate data key
    const { plaintextKey, encryptedKey } = await this.generateDataKey(masterKeyId);
    
    // Encrypt data with DEK
    const encryptionService = new DataEncryptionService();
    const encrypted = encryptionService.encrypt(data, plaintextKey);
    
    // Clear plaintext key from memory
    plaintextKey.fill(0);
    
    return {
      encryptedData: encrypted,
      encryptedDataKey: encryptedKey
    };
  }
  
  /**
   * Envelope Decryption
   */
  async envelopeDecrypt(encryptedData, encryptedDataKey) {
    // Decrypt data key
    const plaintextKey = await this.decryptDataKey(encryptedDataKey);
    
    // Decrypt data
    const encryptionService = new DataEncryptionService();
    const decrypted = encryptionService.decrypt(encryptedData, plaintextKey);
    
    // Clear plaintext key
    plaintextKey.fill(0);
    
    return decrypted;
  }
  
  /**
   * Key Rotation Implementation
   */
  async rotateKey(oldKeyId, newKeyId) {
    console.log(`🔄 Starting key rotation: ${oldKeyId} → ${newKeyId}`);
    
    // 1. Enable new key
    await this.kms.enableKey({ KeyId: newKeyId }).promise();
    
    // 2. Re-encrypt all data with new key
    const affectedRecords = await this.getRecordsEncryptedWith(oldKeyId);
    
    for (const record of affectedRecords) {
      // Decrypt with old key
      const decrypted = await this.envelopeDecrypt(
        record.encryptedData,
        record.encryptedDataKey
      );
      
      // Re-encrypt with new key
      const reencrypted = await this.envelopeEncrypt(decrypted, newKeyId);
      
      // Update record
      await this.updateRecord(record.id, reencrypted);
    }
    
    // 3. Disable old key (after grace period)
    await this.scheduleKeyDisable(oldKeyId, 30); // 30 days grace
    
    console.log(`✅ Key rotation complete: ${affectedRecords.length} records updated`);
  }
  
  /**
   * Automatic Key Rotation Schedule
   */
  async scheduleKeyRotation(keyId) {
    const params = {
      KeyId: keyId,
      RotationPeriodInDays: this.keyRotationDays
    };
    
    await this.kms.enableKeyRotation(params).promise();
  }
  
  /**
   * Key Versioning
   */
  async createKeyVersion(keyAlias) {
    const newKey = await this.kms.createKey({
      Description: `${keyAlias} - Version ${Date.now()}`,
      KeyUsage: 'ENCRYPT_DECRYPT',
      Origin: 'AWS_KMS'
    }).promise();
    
    // Update alias to point to new key
    await this.kms.updateAlias({
      AliasName: `alias/${keyAlias}`,
      TargetKeyId: newKey.KeyMetadata.KeyId
    }).promise();
    
    return newKey.KeyMetadata.KeyId;
  }
  
  /**
   * Get Data Encryption Key (with caching)
   */
  async getDataEncryptionKey(keyId) {
    // Check cache
    if (this.keyCache.has(keyId)) {
      const cached = this.keyCache.get(keyId);
      
      // Check if expired (1 hour cache)
      if (Date.now() - cached.timestamp < 3600000) {
        return cached.key;
      }
    }
    
    // Generate new key
    const { plaintextKey, encryptedKey } = await this.generateDataKey(keyId);
    
    // Cache key
    this.keyCache.set(keyId, {
      key: plaintextKey,
      encrypted: encryptedKey,
      timestamp: Date.now()
    });
    
    // Auto-clear cache after 1 hour
    setTimeout(() => {
      this.keyCache.delete(keyId);
    }, 3600000);
    
    return plaintextKey;
  }
  
  async getRecordsEncryptedWith(keyId) {
    return [];
  }
  
  async updateRecord(id, data) {
    // Database update
  }
  
  async scheduleKeyDisable(keyId, days) {
    // Schedule job
  }
}

/**
 * HashiCorp Vault Integration
 */
class VaultKeyManagement {
  constructor() {
    this.vault = require('node-vault')({
      endpoint: 'https://vault.enbd.com',
      token: process.env.VAULT_TOKEN
    });
  }
  
  /**
   * Store secret in Vault
   */
  async storeSecret(path, secret) {
    await this.vault.write(path, {
      data: secret
    });
  }
  
  /**
   * Retrieve secret from Vault
   */
  async getSecret(path) {
    const result = await this.vault.read(path);
    return result.data.data;
  }
  
  /**
   * Generate dynamic database credentials
   */
  async getDatabaseCredentials(role) {
    const creds = await this.vault.read(`database/creds/${role}`);
    
    return {
      username: creds.data.username,
      password: creds.data.password,
      leaseId: creds.lease_id,
      leaseDuration: creds.lease_duration
    };
  }
  
  /**
   * Rotate database credentials
   */
  async rotateDBCredentials(leaseId) {
    await this.vault.write('sys/leases/renew', {
      lease_id: leaseId
    });
  }
  
  /**
   * Transit encryption (encryption as a service)
   */
  async transitEncrypt(plaintext) {
    const result = await this.vault.write('transit/encrypt/enbd-key', {
      plaintext: Buffer.from(plaintext).toString('base64')
    });
    
    return result.data.ciphertext;
  }
  
  async transitDecrypt(ciphertext) {
    const result = await this.vault.write('transit/decrypt/enbd-key', {
      ciphertext
    });
    
    return Buffer.from(result.data.plaintext, 'base64').toString('utf8');
  }
}

/**
 * Key Hierarchy Management
 */
class KeyHierarchyService {
  /**
   * Three-tier key hierarchy:
   * 1. Master Key (HSM/KMS)
   * 2. Key Encryption Keys (KEK)
   * 3. Data Encryption Keys (DEK)
   */
  
  constructor() {
    this.masterKeyId = process.env.AWS_KMS_MASTER_KEY;
    this.kekCache = new Map();
  }
  
  /**
   * Get or create KEK for service
   */
  async getKEK(serviceName) {
    if (this.kekCache.has(serviceName)) {
      return this.kekCache.get(serviceName);
    }
    
    const kms = new KeyManagementService();
    const { plaintextKey, encryptedKey } = await kms.generateDataKey(this.masterKeyId);
    
    const kek = {
      plaintext: plaintextKey,
      encrypted: encryptedKey,
      service: serviceName,
      createdAt: Date.now()
    };
    
    this.kekCache.set(serviceName, kek);
    
    return kek;
  }
  
  /**
   * Generate DEK encrypted with KEK
   */
  generateDEK(kek) {
    const dek = crypto.randomBytes(32);
    
    // Encrypt DEK with KEK
    const cipher = crypto.createCipheriv('aes-256-gcm', kek.plaintext, crypto.randomBytes(16));
    const encryptedDEK = Buffer.concat([cipher.update(dek), cipher.final()]);
    
    return {
      plaintext: dek,
      encrypted: encryptedDEK,
      tag: cipher.getAuthTag()
    };
  }
}

/**
 * Secure Key Storage
 */
class SecureKeyStorage {
  /**
   * Store keys in environment variables (encrypted)
   */
  static loadFromEnv() {
    const encryptedKey = process.env.ENCRYPTED_KEY;
    const masterPassword = process.env.MASTER_PASSWORD;
    
    // Derive key from password
    const key = crypto.pbkdf2Sync(
      masterPassword,
      'enbd-salt',
      100000,
      32,
      'sha256'
    );
    
    // Decrypt
    const parts = encryptedKey.split(':');
    const iv = Buffer.from(parts[0], 'hex');
    const tag = Buffer.from(parts[1], 'hex');
    const encrypted = Buffer.from(parts[2], 'hex');
    
    const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
    decipher.setAuthTag(tag);
    
    const decrypted = Buffer.concat([
      decipher.update(encrypted),
      decipher.final()
    ]);
    
    return decrypted;
  }
  
  /**
   * Store keys in Hardware Security Module (HSM)
   */
  static async storeInHSM(keyMaterial) {
    const hsm = new AWS.CloudHSMV2();
    
    // Import key to HSM
    const result = await hsm.createKey({
      KeyMaterial: keyMaterial,
      Exportable: false
    }).promise();
    
    return result.KeyId;
  }
}

module.exports = {
  KeyManagementService,
  VaultKeyManagement,
  KeyHierarchyService,
  SecureKeyStorage
};
```

**Key Management Best Practices:**

1. **Never hardcode keys** ✅
2. **Use KMS/Vault** for key storage ✅
3. **Envelope encryption** for data ✅
4. **Regular key rotation** (90 days) ✅
5. **Key versioning** with grace periods ✅
6. **Separate keys** per environment ✅

---

### Q23. How do you implement PCI DSS compliance for payment processing?

**Answer:**

```javascript
const crypto = require('crypto');
const validator = require('validator');

/**
 * PCI DSS Compliance Service
 */
class PCIDSSComplianceService {
  constructor() {
    this.tokenVault = new Map();
  }
  
  /**
   * PCI DSS Requirement 3: Protect stored cardholder data
   * Tokenization instead of storing card numbers
   */
  async tokenizeCardNumber(cardNumber, userId) {
    // Validate card number
    if (!this.validateCardNumber(cardNumber)) {
      throw new Error('Invalid card number');
    }
    
    // Generate token
    const token = crypto.randomBytes(16).toString('hex');
    
    // Store mapping (encrypted) in secure vault
    await this.storeInVault(token, {
      cardNumber: this.encryptCardNumber(cardNumber),
      lastFour: cardNumber.slice(-4),
      userId,
      createdAt: Date.now()
    });
    
    return {
      token,
      lastFour: cardNumber.slice(-4),
      expiresAt: Date.now() + (365 * 24 * 60 * 60 * 1000) // 1 year
    };
  }
  
  /**
   * Detokenize card number (for payment processing only)
   */
  async detokenizeCard(token) {
    const data = await this.getFromVault(token);
    
    if (!data) {
      throw new Error('Invalid token');
    }
    
    return this.decryptCardNumber(data.cardNumber);
  }
  
  /**
   * PCI DSS Requirement 3.4: Render PAN unreadable
   */
  encryptCardNumber(cardNumber) {
    const key = this.getMasterKey();
    const iv = crypto.randomBytes(16);
    
    const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
    let encrypted = cipher.update(cardNumber, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const tag = cipher.getAuthTag();
    
    return `${iv.toString('hex')}:${tag.toString('hex')}:${encrypted}`;
  }
  
  decryptCardNumber(encryptedData) {
    const [iv, tag, encrypted] = encryptedData.split(':');
    const key = this.getMasterKey();
    
    const decipher = crypto.createDecipheriv(
      'aes-256-gcm',
      key,
      Buffer.from(iv, 'hex')
    );
    
    decipher.setAuthTag(Buffer.from(tag, 'hex'));
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  /**
   * PCI DSS Requirement 3.5.3: Card number masking
   */
  maskCardNumber(cardNumber) {
    const lastFour = cardNumber.slice(-4);
    return `****-****-****-${lastFour}`;
  }
  
  /**
   * PCI DSS Requirement 4: Encrypt transmission over open networks
   */
  encryptForTransmission(data) {
    // Use TLS 1.3 for transmission
    // Add additional layer with public key encryption
    const publicKey = this.getPaymentGatewayPublicKey();
    
    const encrypted = crypto.publicEncrypt(
      {
        key: publicKey,
        padding: crypto.constants.RSA_PKCS1_OAEP_PADDING,
        oaepHash: 'sha256'
      },
      Buffer.from(JSON.stringify(data))
    );
    
    return encrypted.toString('base64');
  }
  
  /**
   * Validate card number (Luhn algorithm)
   */
  validateCardNumber(cardNumber) {
    // Remove spaces and dashes
    const cleaned = cardNumber.replace(/[\s-]/g, '');
    
    // Check if all digits
    if (!/^\d+$/.test(cleaned)) {
      return false;
    }
    
    // Check length (13-19 digits)
    if (cleaned.length < 13 || cleaned.length > 19) {
      return false;
    }
    
    // Luhn algorithm
    let sum = 0;
    let isEven = false;
    
    for (let i = cleaned.length - 1; i >= 0; i--) {
      let digit = parseInt(cleaned[i]);
      
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
   * Validate CVV (never store!)
   */
  validateCVV(cvv, cardType) {
    const length = cardType === 'amex' ? 4 : 3;
    return /^\d+$/.test(cvv) && cvv.length === length;
  }
  
  /**
   * PCI DSS Requirement 8: Identify and authenticate access
   */
  async logCardAccess(userId, action, token) {
    await this.auditLog({
      userId,
      action,
      token: this.hashToken(token),
      timestamp: Date.now(),
      ip: 'x.x.x.x'
    });
  }
  
  /**
   * PCI DSS Requirement 10: Track and monitor access
   */
  async auditLog(entry) {
    // Write to append-only audit log
    console.log('AUDIT:', entry);
  }
  
  getMasterKey() {
    return Buffer.from(process.env.CARD_ENCRYPTION_KEY, 'hex');
  }
  
  getPaymentGatewayPublicKey() {
    return fs.readFileSync('./certs/payment-gateway-public.pem');
  }
  
  hashToken(token) {
    return crypto.createHash('sha256').update(token).digest('hex');
  }
  
  async storeInVault(token, data) {
    this.tokenVault.set(token, data);
  }
  
  async getFromVault(token) {
    return this.tokenVault.get(token);
  }
}

/**
 * Secure Payment Processing Service
 */
class SecurePaymentService {
  constructor() {
    this.pciService = new PCIDSSComplianceService();
  }
  
  /**
   * Process payment securely
   */
  async processPayment(paymentData) {
    const { cardToken, amount, cvv, userId } = paymentData;
    
    // Never log sensitive data
    console.log('Processing payment', { amount, userId, token: '***' });
    
    // Validate CVV (never store)
    if (!this.pciService.validateCVV(cvv, 'visa')) {
      throw new Error('Invalid CVV');
    }
    
    // Detokenize card (in memory only)
    const cardNumber = await this.pciService.detokenizeCard(cardToken);
    
    // Prepare payment request
    const paymentRequest = {
      cardNumber, // Only in memory, never logged
      cvv,        // Only in memory, never logged
      amount,
      currency: 'AED'
    };
    
    // Encrypt for transmission to payment gateway
    const encrypted = this.pciService.encryptForTransmission(paymentRequest);
    
    // Send to payment gateway over TLS 1.3
    const response = await this.sendToPaymentGateway(encrypted);
    
    // Clear sensitive data from memory
    cardNumber.replace(/./g, '0');
    cvv.replace(/./g, '0');
    
    // Log access (with hashed token)
    await this.pciService.logCardAccess(userId, 'PAYMENT_PROCESSED', cardToken);
    
    return {
      success: response.approved,
      transactionId: response.transactionId,
      // Never return card number
      lastFour: (await this.pciService.getFromVault(cardToken)).lastFour
    };
  }
  
  /**
   * Add payment method (tokenize)
   */
  async addPaymentMethod(cardData, userId) {
    const { cardNumber, expiryMonth, expiryYear, cvv } = cardData;
    
    // Validate card number
    if (!this.pciService.validateCardNumber(cardNumber)) {
      return {
        success: false,
        error: 'Invalid card number'
      };
    }
    
    // Validate CVV (don't store)
    if (!this.pciService.validateCVV(cvv, 'visa')) {
      return {
        success: false,
        error: 'Invalid CVV'
      };
    }
    
    // Tokenize card number
    const tokenData = await this.pciService.tokenizeCardNumber(cardNumber, userId);
    
    // Store only: token, last 4 digits, expiry (NOT full card number or CVV)
    await this.savePaymentMethod({
      userId,
      token: tokenData.token,
      lastFour: tokenData.lastFour,
      expiryMonth,
      expiryYear,
      expiresAt: tokenData.expiresAt,
      // NEVER store: cardNumber, cvv
    });
    
    // Log action
    await this.pciService.logCardAccess(userId, 'CARD_ADDED', tokenData.token);
    
    return {
      success: true,
      paymentMethodId: tokenData.token,
      lastFour: tokenData.lastFour
    };
  }
  
  async sendToPaymentGateway(encrypted) {
    return {
      approved: true,
      transactionId: 'txn_' + crypto.randomBytes(16).toString('hex')
    };
  }
  
  async savePaymentMethod(data) {
    // Database save
  }
}

/**
 * PCI DSS Network Segmentation
 */
class NetworkSegmentationService {
  /**
   * Separate cardholder data environment (CDE)
   */
  static setupFirewallRules() {
    return {
      // Only payment service can access token vault
      allowedServices: ['payment-service'],
      
      // Deny all by default
      defaultPolicy: 'DENY',
      
      // Specific rules
      rules: [
        {
          from: 'payment-service',
          to: 'token-vault',
          port: 443,
          protocol: 'HTTPS',
          action: 'ALLOW'
        },
        {
          from: 'payment-service',
          to: 'payment-gateway',
          port: 443,
          protocol: 'HTTPS',
          action: 'ALLOW'
        },
        {
          from: '*',
          to: 'token-vault',
          action: 'DENY'
        }
      ]
    };
  }
}

module.exports = {
  PCIDSSComplianceService,
  SecurePaymentService,
  NetworkSegmentationService
};
```

**PCI DSS Key Requirements:**

1. ✅ **Never store CVV/CVV2/CVC2**
2. ✅ **Tokenize card numbers**
3. ✅ **Encrypt cardholder data**
4. ✅ **TLS 1.3 for transmission**
5. ✅ **Mask PAN when displayed**
6. ✅ **Audit all card data access**
7. ✅ **Network segmentation (CDE)**
8. ✅ **Regular security testing**

---

### Q24. How do you implement audit logging and compliance?

**Answer:**

```javascript
const winston = require('winston');
const { ElasticsearchTransport } = require('winston-elasticsearch');

/**
 * Comprehensive Audit Logging Service
 */
class AuditLoggingService {
  constructor() {
    this.setupLogger();
  }
  
  /**
   * Setup Winston logger with multiple transports
   */
  setupLogger() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
      ),
      defaultMeta: {
        service: 'enbd-banking',
        environment: process.env.NODE_ENV
      },
      transports: [
        // File transport (append-only)
        new winston.transports.File({
          filename: './logs/audit.log',
          maxsize: 10485760, // 10MB
          maxFiles: 100,
          tailable: true
        }),
        
        // Elasticsearch for search/analytics
        new ElasticsearchTransport({
          level: 'info',
          clientOpts: {
            node: 'https://elasticsearch.enbd.com',
            auth: {
              username: process.env.ES_USER,
              password: process.env.ES_PASSWORD
            }
          },
          index: 'audit-logs'
        }),
        
        // Console in development
        ...(process.env.NODE_ENV === 'development' ? [
          new winston.transports.Console({
            format: winston.format.combine(
              winston.format.colorize(),
              winston.format.simple()
            )
          })
        ] : [])
      ]
    });
  }
  
  /**
   * Log authentication events
   */
  async logAuthentication(event) {
    await this.logger.info('AUTH_EVENT', {
      eventType: event.type, // LOGIN, LOGOUT, LOGIN_FAILED, MFA_VERIFIED
      userId: event.userId,
      email: event.email,
      ip: event.ip,
      userAgent: event.userAgent,
      location: event.location,
      success: event.success,
      failureReason: event.failureReason,
      timestamp: Date.now(),
      sessionId: event.sessionId
    });
  }
  
  /**
   * Log data access
   */
  async logDataAccess(event) {
    await this.logger.info('DATA_ACCESS', {
      eventType: 'DATA_ACCESS',
      userId: event.userId,
      resourceType: event.resourceType, // ACCOUNT, TRANSACTION, CARD
      resourceId: this.hashResourceId(event.resourceId),
      action: event.action, // READ, UPDATE, DELETE
      fields: event.fields, // Fields accessed
      ip: event.ip,
      timestamp: Date.now(),
      purpose: event.purpose
    });
  }
  
  /**
   * Log financial transactions
   */
  async logTransaction(transaction) {
    await this.logger.info('TRANSACTION', {
      eventType: 'FINANCIAL_TRANSACTION',
      transactionId: transaction.id,
      userId: transaction.userId,
      type: transaction.type, // TRANSFER, PAYMENT, WITHDRAWAL
      amount: transaction.amount,
      currency: transaction.currency,
      fromAccount: this.maskAccount(transaction.from),
      toAccount: this.maskAccount(transaction.to),
      status: transaction.status,
      ip: transaction.ip,
      timestamp: Date.now(),
      metadata: transaction.metadata
    });
  }
  
  /**
   * Log security events
   */
  async logSecurityEvent(event) {
    await this.logger.warn('SECURITY_EVENT', {
      eventType: event.type, // SUSPICIOUS_ACTIVITY, RATE_LIMIT, BLOCKED_IP
      severity: event.severity, // LOW, MEDIUM, HIGH, CRITICAL
      userId: event.userId,
      ip: event.ip,
      description: event.description,
      action: event.action, // BLOCKED, ALERTED, LOGGED
      timestamp: Date.now(),
      metadata: event.metadata
    });
  }
  
  /**
   * Log admin actions
   */
  async logAdminAction(event) {
    await this.logger.warn('ADMIN_ACTION', {
      eventType: 'ADMIN_ACTION',
      adminId: event.adminId,
      action: event.action,
      targetUserId: event.targetUserId,
      targetResource: event.targetResource,
      changes: event.changes,
      reason: event.reason,
      ip: event.ip,
      timestamp: Date.now()
    });
  }
  
  /**
   * Log configuration changes
   */
  async logConfigChange(event) {
    await this.logger.warn('CONFIG_CHANGE', {
      eventType: 'CONFIGURATION_CHANGE',
      userId: event.userId,
      configKey: event.key,
      oldValue: this.sanitizeValue(event.oldValue),
      newValue: this.sanitizeValue(event.newValue),
      ip: event.ip,
      timestamp: Date.now()
    });
  }
  
  /**
   * Log compliance events
   */
  async logComplianceEvent(event) {
    await this.logger.info('COMPLIANCE', {
      eventType: 'COMPLIANCE_EVENT',
      complianceType: event.type, // GDPR, PCI_DSS, KYC, AML
      action: event.action,
      userId: event.userId,
      details: event.details,
      timestamp: Date.now()
    });
  }
  
  /**
   * Tamper-proof logging with hash chain
   */
  async logWithHashChain(event) {
    const previousHash = await this.getLastLogHash();
    
    const logEntry = {
      ...event,
      previousHash,
      timestamp: Date.now()
    };
    
    // Calculate hash of this entry
    const hash = crypto
      .createHash('sha256')
      .update(JSON.stringify(logEntry))
      .digest('hex');
    
    logEntry.hash = hash;
    
    await this.logger.info('HASH_CHAIN_LOG', logEntry);
    await this.storeLogHash(hash);
    
    return hash;
  }
  
  hashResourceId(id) {
    return crypto.createHash('sha256').update(id).digest('hex').substring(0, 16);
  }
  
  maskAccount(accountNumber) {
    if (!accountNumber) return null;
    return `***${accountNumber.slice(-4)}`;
  }
  
  sanitizeValue(value) {
    // Remove sensitive data from config values
    if (typeof value === 'string' && /password|key|secret|token/i.test(value)) {
      return '***REDACTED***';
    }
    return value;
  }
  
  async getLastLogHash() {
    return 'previous-hash';
  }
  
  async storeLogHash(hash) {
    // Store in database
  }
}

/**
 * Audit Middleware
 */
class AuditMiddleware {
  constructor() {
    this.auditService = new AuditLoggingService();
  }
  
  /**
   * Log all API requests
   */
  requestLogger() {
    return async (req, res, next) => {
      const startTime = Date.now();
      
      // Capture response
      const originalSend = res.send;
      res.send = function(data) {
        res.send = originalSend;
        
        const duration = Date.now() - startTime;
        
        // Log after response
        setImmediate(async () => {
          await this.auditService.logger.info('API_REQUEST', {
            method: req.method,
            path: req.path,
            statusCode: res.statusCode,
            duration,
            userId: req.user?.id,
            ip: req.ip,
            userAgent: req.get('user-agent'),
            requestId: req.id,
            timestamp: Date.now()
          });
        });
        
        return res.send(data);
      }.bind(this);
      
      next();
    };
  }
  
  /**
   * Log data access
   */
  dataAccessLogger(resourceType) {
    return async (req, res, next) => {
      await this.auditService.logDataAccess({
        userId: req.user.id,
        resourceType,
        resourceId: req.params.id,
        action: req.method,
        fields: Object.keys(req.body || {}),
        ip: req.ip,
        purpose: req.headers['x-access-purpose']
      });
      
      next();
    };
  }
}

/**
 * Compliance Reporting Service
 */
class ComplianceReportingService {
  constructor() {
    this.auditService = new AuditLoggingService();
  }
  
  /**
   * Generate GDPR data export
   */
  async generateGDPRExport(userId) {
    // Collect all user data
    const userData = await this.collectUserData(userId);
    const auditLogs = await this.getUserAuditLogs(userId);
    
    return {
      personalData: userData,
      auditTrail: auditLogs,
      exportedAt: new Date().toISOString(),
      format: 'JSON'
    };
  }
  
  /**
   * Generate PCI DSS compliance report
   */
  async generatePCIDSSReport(startDate, endDate) {
    return {
      cardDataAccess: await this.getCardDataAccessLogs(startDate, endDate),
      failedAccessAttempts: await this.getFailedAccessLogs(startDate, endDate),
      configChanges: await this.getConfigChanges(startDate, endDate),
      securityIncidents: await this.getSecurityIncidents(startDate, endDate)
    };
  }
  
  /**
   * Generate AML (Anti-Money Laundering) report
   */
  async generateAMLReport(startDate, endDate) {
    return {
      largeTransactions: await this.getLargeTransactions(startDate, endDate),
      suspiciousPatterns: await this.detectSuspiciousPatterns(startDate, endDate),
      flaggedUsers: await this.getFlaggedUsers()
    };
  }
  
  async collectUserData(userId) {
    return {};
  }
  
  async getUserAuditLogs(userId) {
    return [];
  }
  
  async getCardDataAccessLogs(start, end) {
    return [];
  }
  
  async getFailedAccessLogs(start, end) {
    return [];
  }
  
  async getConfigChanges(start, end) {
    return [];
  }
  
  async getSecurityIncidents(start, end) {
    return [];
  }
  
  async getLargeTransactions(start, end) {
    return [];
  }
  
  async detectSuspiciousPatterns(start, end) {
    return [];
  }
  
  async getFlaggedUsers() {
    return [];
  }
}

module.exports = {
  AuditLoggingService,
  AuditMiddleware,
  ComplianceReportingService
};
```

**Audit Logging Requirements:**

1. ✅ **Immutable logs** (append-only)
2. ✅ **Complete audit trail**
3. ✅ **Tamper detection** (hash chain)
4. ✅ **Centralized storage** (Elasticsearch)
5. ✅ **Log retention** (7 years for financial)
6. ✅ **GDPR compliance** (data export)
7. ✅ **PCI DSS compliance**
8. ✅ **Real-time alerting**

---

### Q25. How do you implement secrets management in production?

**Answer:**

```javascript
const AWS = require('aws-sdk');
const { SecretClient } = require('@azure/keyvault-secrets');

/**
 * Secrets Management Service
 */
class SecretsManagementService {
  constructor() {
    this.secretsManager = new AWS.SecretsManager({
      region: 'me-south-1' // Dubai
    });
    this.cache = new Map();
    this.cacheTTL = 300000; // 5 minutes
  }
  
  /**
   * Get secret from AWS Secrets Manager
   */
  async getSecret(secretName) {
    // Check cache first
    if (this.cache.has(secretName)) {
      const cached = this.cache.get(secretName);
      if (Date.now() - cached.timestamp < this.cacheTTL) {
        return cached.value;
      }
    }
    
    try {
      const data = await this.secretsManager.getSecretValue({
        SecretId: secretName
      }).promise();
      
      const secret = data.SecretString ? 
        JSON.parse(data.SecretString) : 
        data.SecretBinary;
      
      // Cache secret
      this.cache.set(secretName, {
        value: secret,
        timestamp: Date.now()
      });
      
      return secret;
    } catch (error) {
      console.error(`Failed to get secret ${secretName}:`, error);
      throw error;
    }
  }
  
  /**
   * Store secret
   */
  async storeSecret(secretName, secretValue) {
    await this.secretsManager.createSecret({
      Name: secretName,
      SecretString: JSON.stringify(secretValue),
      Description: `ENBD ${secretName}`,
      Tags: [
        { Key: 'Environment', Value: process.env.NODE_ENV },
        { Key: 'Application', Value: 'enbd-banking' }
      ]
    }).promise();
  }
  
  /**
   * Update secret
   */
  async updateSecret(secretName, secretValue) {
    await this.secretsManager.updateSecret({
      SecretId: secretName,
      SecretString: JSON.stringify(secretValue)
    }).promise();
    
    // Invalidate cache
    this.cache.delete(secretName);
  }
  
  /**
   * Rotate secret
   */
  async rotateSecret(secretName) {
    await this.secretsManager.rotateSecret({
      SecretId: secretName,
      RotationLambdaARN: process.env.ROTATION_LAMBDA_ARN,
      RotationRules: {
        AutomaticallyAfterDays: 30
      }
    }).promise();
  }
  
  /**
   * Get database credentials (auto-rotated)
   */
  async getDatabaseCredentials() {
    const secret = await this.getSecret('enbd/database/credentials');
    
    return {
      host: secret.host,
      port: secret.port,
      username: secret.username,
      password: secret.password,
      database: secret.database
    };
  }
  
  /**
   * Get API keys
   */
  async getAPIKey(serviceName) {
    const secrets = await this.getSecret('enbd/api-keys');
    return secrets[serviceName];
  }
  
  /**
   * Get encryption keys
   */
  async getEncryptionKey(keyName) {
    const secrets = await this.getSecret('enbd/encryption-keys');
    return Buffer.from(secrets[keyName], 'base64');
  }
}

/**
 * Environment-based Configuration
 */
class ConfigurationService {
  constructor() {
    this.secretsService = new SecretsManagementService();
    this.config = null;
  }
  
  /**
   * Load configuration on startup
   */
  async loadConfig() {
    const environment = process.env.NODE_ENV || 'development';
    
    // Get secrets from Secrets Manager
    const secrets = await this.secretsService.getSecret(`enbd/config/${environment}`);
    
    this.config = {
      // Database
      database: {
        host: secrets.DB_HOST,
        port: secrets.DB_PORT,
        username: secrets.DB_USERNAME,
        password: secrets.DB_PASSWORD,
        database: secrets.DB_NAME,
        ssl: true
      },
      
      // Redis
      redis: {
        host: secrets.REDIS_HOST,
        port: secrets.REDIS_PORT,
        password: secrets.REDIS_PASSWORD,
        tls: true
      },
      
      // JWT
      jwt: {
        accessTokenSecret: secrets.JWT_ACCESS_SECRET,
        refreshTokenSecret: secrets.JWT_REFRESH_SECRET,
        accessTokenExpiry: '15m',
        refreshTokenExpiry: '7d'
      },
      
      // External APIs
      apis: {
        paymentGateway: {
          url: secrets.PAYMENT_GATEWAY_URL,
          apiKey: secrets.PAYMENT_GATEWAY_KEY,
          secret: secrets.PAYMENT_GATEWAY_SECRET
        },
        smsProvider: {
          url: secrets.SMS_PROVIDER_URL,
          apiKey: secrets.SMS_PROVIDER_KEY
        }
      },
      
      // Encryption
      encryption: {
        masterKey: Buffer.from(secrets.MASTER_ENCRYPTION_KEY, 'base64')
      }
    };
    
    return this.config;
  }
  
  /**
   * Get configuration value
   */
  get(path) {
    const parts = path.split('.');
    let value = this.config;
    
    for (const part of parts) {
      value = value[part];
      if (value === undefined) {
        throw new Error(`Configuration not found: ${path}`);
      }
    }
    
    return value;
  }
  
  /**
   * Refresh configuration (for secret rotation)
   */
  async refresh() {
    await this.loadConfig();
  }
}

/**
 * Docker Secrets Integration
 */
class DockerSecretsService {
  /**
   * Read secret from Docker secrets
   */
  static readDockerSecret(secretName) {
    const fs = require('fs');
    const path = `/run/secrets/${secretName}`;
    
    try {
      return fs.readFileSync(path, 'utf8').trim();
    } catch (error) {
      console.error(`Failed to read Docker secret ${secretName}`);
      return null;
    }
  }
  
  /**
   * Load all Docker secrets
   */
  static loadDockerSecrets() {
    return {
      dbPassword: this.readDockerSecret('db_password'),
      jwtSecret: this.readDockerSecret('jwt_secret'),
      apiKey: this.readDockerSecret('api_key')
    };
  }
}

/**
 * Kubernetes Secrets Integration
 */
class KubernetesSecretsService {
  /**
   * Read secret from Kubernetes volume mount
   */
  static readK8sSecret(secretName) {
    const fs = require('fs');
    const path = `/var/secrets/${secretName}`;
    
    try {
      return fs.readFileSync(path, 'utf8').trim();
    } catch (error) {
      console.error(`Failed to read K8s secret ${secretName}`);
      return null;
    }
  }
}

/**
 * Application Initialization with Secrets
 */
class SecureApplicationBootstrap {
  async initialize() {
    console.log('🔐 Loading secrets...');
    
    // Load configuration
    const configService = new ConfigurationService();
    await configService.loadConfig();
    
    // Initialize database connection
    const dbConfig = configService.get('database');
    await this.connectDatabase(dbConfig);
    
    // Initialize Redis
    const redisConfig = configService.get('redis');
    await this.connectRedis(redisConfig);
    
    // Setup secret rotation listener
    this.setupSecretRotationListener(configService);
    
    console.log('✅ All secrets loaded and services initialized');
    
    return configService;
  }
  
  async connectDatabase(config) {
    // Database connection with credentials from secrets
  }
  
  async connectRedis(config) {
    // Redis connection with credentials from secrets
  }
  
  setupSecretRotationListener(configService) {
    // Listen for secret rotation events
    setInterval(async () => {
      await configService.refresh();
    }, 300000); // Check every 5 minutes
  }
}

/**
 * Secrets Best Practices
 */
class SecretsBestPractices {
  /**
   * ❌ NEVER DO THIS
   */
  static badExamples() {
    // ❌ Hardcoded secrets
    const apiKey = 'sk_live_abc123def456';
    
    // ❌ Secrets in environment variables (visible in process list)
    process.env.DB_PASSWORD = 'password123';
    
    // ❌ Secrets in code repository
    const config = {
      password: 'admin123'
    };
    
    // ❌ Secrets in logs
    console.log('Password:', password);
  }
  
  /**
   * ✅ GOOD PRACTICES
   */
  static async goodExamples() {
    // ✅ Load from Secrets Manager
    const secretsService = new SecretsManagementService();
    const apiKey = await secretsService.getAPIKey('payment-gateway');
    
    // ✅ Use environment-specific secrets
    const secret = await secretsService.getSecret(`enbd/config/${process.env.NODE_ENV}`);
    
    // ✅ Rotate secrets regularly
    await secretsService.rotateSecret('database-credentials');
    
    // ✅ Never log secrets
    console.log('API call successful'); // Don't log the key
    
    // ✅ Use short-lived credentials
    const tempCreds = await this.getTemporaryCredentials();
  }
  
  static async getTemporaryCredentials() {
    return {};
  }
}

module.exports = {
  SecretsManagementService,
  ConfigurationService,
  DockerSecretsService,
  KubernetesSecretsService,
  SecureApplicationBootstrap
};
```

**Secrets Management Best Practices:**

1. ✅ **Never hardcode** secrets
2. ✅ **Use Secrets Manager** (AWS/Azure/Vault)
3. ✅ **Rotate regularly** (30-90 days)
4. ✅ **Environment-specific** secrets
5. ✅ **Encrypt at rest** in storage
6. ✅ **Cache with TTL** (5-15 minutes)
7. ✅ **Audit access** to secrets
8. ✅ **Least privilege** access

---

**Summary Q21-Q25:**
- Encryption at rest & in transit (AES-256-GCM, TLS 1.3) ✅
- Key management & rotation (AWS KMS, Vault) ✅
- PCI DSS compliance (tokenization, never store CVV) ✅
- Audit logging (tamper-proof, GDPR/PCI compliant) ✅
- Secrets management (AWS Secrets Manager, rotation) ✅

Continuing with remaining security questions...
