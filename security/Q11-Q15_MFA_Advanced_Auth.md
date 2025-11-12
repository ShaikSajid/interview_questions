# Security Questions 11-15: Multi-Factor Authentication & Advanced Auth

---

### Q11. How do you implement Multi-Factor Authentication (MFA) with TOTP?

**Answer:**

```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');
const crypto = require('crypto');

/**
 * TOTP-based MFA Service for ENBD
 */
class TOTPMFAService {
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT
    });
  }
  
  /**
   * Generate TOTP secret for user
   */
  async enableMFA(userId, userEmail) {
    // Generate secret
    const secret = speakeasy.generateSecret({
      name: `ENBD Banking (${userEmail})`,
      issuer: 'Emirates NBD',
      length: 32
    });
    
    // Store secret (encrypted)
    const encryptedSecret = this.encryptSecret(secret.base32);
    await this.redis.hset(`user:${userId}:mfa`, {
      secret: encryptedSecret,
      enabled: 'false',
      backupCodes: JSON.stringify(this.generateBackupCodes())
    });
    
    // Generate QR code
    const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url);
    
    return {
      secret: secret.base32,
      qrCode: qrCodeUrl,
      manualEntry: secret.base32 // For manual entry
    };
  }
  
  /**
   * Verify TOTP token and enable MFA
   */
  async verifyAndEnableMFA(userId, token) {
    const mfaData = await this.redis.hgetall(`user:${userId}:mfa`);
    
    if (!mfaData || !mfaData.secret) {
      throw new Error('MFA not initialized');
    }
    
    const secret = this.decryptSecret(mfaData.secret);
    
    // Verify token
    const verified = speakeasy.totp.verify({
      secret,
      encoding: 'base32',
      token,
      window: 2 // Allow 2 time steps (60 seconds) tolerance
    });
    
    if (!verified) {
      return { success: false, error: 'Invalid token' };
    }
    
    // Enable MFA
    await this.redis.hset(`user:${userId}:mfa`, 'enabled', 'true');
    
    // Get backup codes
    const backupCodes = JSON.parse(mfaData.backupCodes);
    
    return {
      success: true,
      backupCodes // Show once during setup
    };
  }
  
  /**
   * Verify TOTP token during login
   */
  async verifyMFAToken(userId, token) {
    const mfaData = await this.redis.hgetall(`user:${userId}:mfa`);
    
    if (!mfaData || mfaData.enabled !== 'true') {
      return { success: false, error: 'MFA not enabled' };
    }
    
    const secret = this.decryptSecret(mfaData.secret);
    
    // Verify TOTP token
    const verified = speakeasy.totp.verify({
      secret,
      encoding: 'base32',
      token,
      window: 2
    });
    
    if (verified) {
      return { success: true };
    }
    
    // Check if it's a backup code
    const backupCodes = JSON.parse(mfaData.backupCodes);
    const backupIndex = backupCodes.indexOf(token);
    
    if (backupIndex !== -1) {
      // Remove used backup code
      backupCodes.splice(backupIndex, 1);
      await this.redis.hset(
        `user:${userId}:mfa`,
        'backupCodes',
        JSON.stringify(backupCodes)
      );
      
      return { success: true, usedBackupCode: true };
    }
    
    return { success: false, error: 'Invalid token' };
  }
  
  /**
   * Disable MFA
   */
  async disableMFA(userId, password) {
    // Verify password before disabling
    const user = await this.getUserById(userId);
    const passwordValid = await this.verifyPassword(password, user.passwordHash);
    
    if (!passwordValid) {
      throw new Error('Invalid password');
    }
    
    await this.redis.del(`user:${userId}:mfa`);
    
    return { success: true };
  }
  
  /**
   * Generate backup codes
   */
  generateBackupCodes(count = 10) {
    const codes = [];
    for (let i = 0; i < count; i++) {
      const code = crypto.randomBytes(4).toString('hex').toUpperCase();
      codes.push(code);
    }
    return codes;
  }
  
  /**
   * Regenerate backup codes
   */
  async regenerateBackupCodes(userId) {
    const newCodes = this.generateBackupCodes();
    
    await this.redis.hset(
      `user:${userId}:mfa`,
      'backupCodes',
      JSON.stringify(newCodes)
    );
    
    return newCodes;
  }
  
  encryptSecret(secret) {
    const cipher = crypto.createCipher('aes-256-cbc', process.env.MFA_ENCRYPTION_KEY);
    let encrypted = cipher.update(secret, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
  }
  
  decryptSecret(encrypted) {
    const decipher = crypto.createDecipher('aes-256-cbc', process.env.MFA_ENCRYPTION_KEY);
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
  }
  
  async getUserById(userId) {
    // Database lookup
    return { id: userId, passwordHash: 'hash' };
  }
  
  async verifyPassword(password, hash) {
    return true; // bcrypt.compare(password, hash)
  }
}

/**
 * SMS-based MFA Service
 */
class SMSMFAService {
  constructor() {
    this.redis = new Redis();
    this.twilioClient = require('twilio')(
      process.env.TWILIO_ACCOUNT_SID,
      process.env.TWILIO_AUTH_TOKEN
    );
  }
  
  /**
   * Send SMS verification code
   */
  async sendVerificationCode(userId, phoneNumber) {
    // Generate 6-digit code
    const code = crypto.randomInt(100000, 999999).toString();
    
    // Store code with expiration (5 minutes)
    await this.redis.setex(
      `mfa:sms:${userId}`,
      5 * 60,
      JSON.stringify({
        code,
        phoneNumber,
        attempts: 0
      })
    );
    
    // Send SMS
    try {
      await this.twilioClient.messages.create({
        body: `Your ENBD verification code is: ${code}. Valid for 5 minutes.`,
        from: process.env.TWILIO_PHONE_NUMBER,
        to: phoneNumber
      });
      
      return { success: true };
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
  
  /**
   * Verify SMS code
   */
  async verifyCode(userId, code) {
    const stored = await this.redis.get(`mfa:sms:${userId}`);
    
    if (!stored) {
      return { success: false, error: 'Code expired or not found' };
    }
    
    const data = JSON.parse(stored);
    
    // Check attempts
    if (data.attempts >= 3) {
      await this.redis.del(`mfa:sms:${userId}`);
      return { success: false, error: 'Too many attempts' };
    }
    
    // Verify code
    if (data.code === code) {
      await this.redis.del(`mfa:sms:${userId}`);
      return { success: true };
    }
    
    // Increment attempts
    data.attempts++;
    await this.redis.setex(
      `mfa:sms:${userId}`,
      5 * 60,
      JSON.stringify(data)
    );
    
    return {
      success: false,
      error: 'Invalid code',
      attemptsRemaining: 3 - data.attempts
    };
  }
}

/**
 * Complete MFA Authentication Flow
 */
class MFAAuthenticationController {
  constructor() {
    this.totpService = new TOTPMFAService();
    this.smsService = new SMSMFAService();
    this.jwtService = new JWTService();
  }
  
  setupRoutes(app) {
    // Step 1: Regular login
    app.post('/api/auth/login', this.login.bind(this));
    
    // Step 2: Verify MFA
    app.post('/api/auth/mfa/verify', this.verifyMFA.bind(this));
    
    // MFA setup endpoints
    app.post('/api/user/mfa/enable', this.enableMFA.bind(this));
    app.post('/api/user/mfa/verify-setup', this.verifyMFASetup.bind(this));
    app.post('/api/user/mfa/disable', this.disableMFA.bind(this));
    app.post('/api/user/mfa/backup-codes', this.regenerateBackupCodes.bind(this));
  }
  
  async login(req, res) {
    const { email, password } = req.body;
    
    // Step 1: Verify credentials
    const user = await this.authenticateUser(email, password);
    
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Step 2: Check if MFA is enabled
    const mfaData = await this.totpService.redis.hgetall(`user:${user.id}:mfa`);
    
    if (mfaData && mfaData.enabled === 'true') {
      // MFA required - generate temporary token
      const tempToken = this.jwtService.generateTempToken(user.id);
      
      return res.json({
        mfaRequired: true,
        tempToken,
        mfaMethods: ['totp', 'sms']
      });
    }
    
    // No MFA - return access token
    const accessToken = this.jwtService.generateAccessToken(user);
    
    res.json({
      accessToken,
      tokenType: 'Bearer',
      expiresIn: 900
    });
  }
  
  async verifyMFA(req, res) {
    const { tempToken, method, code } = req.body;
    
    // Verify temp token
    const tempDecoded = this.jwtService.verifyTempToken(tempToken);
    
    if (!tempDecoded.valid) {
      return res.status(401).json({ error: 'Invalid or expired token' });
    }
    
    const userId = tempDecoded.userId;
    
    // Verify MFA code
    let verified;
    
    if (method === 'totp') {
      verified = await this.totpService.verifyMFAToken(userId, code);
    } else if (method === 'sms') {
      verified = await this.smsService.verifyCode(userId, code);
    } else {
      return res.status(400).json({ error: 'Invalid MFA method' });
    }
    
    if (!verified.success) {
      return res.status(401).json({ error: verified.error });
    }
    
    // MFA verified - generate access token
    const user = await this.getUserById(userId);
    const accessToken = this.jwtService.generateAccessToken(user);
    
    res.json({
      accessToken,
      tokenType: 'Bearer',
      expiresIn: 900,
      usedBackupCode: verified.usedBackupCode
    });
  }
  
  async enableMFA(req, res) {
    const userId = req.user.id; // From auth middleware
    const { email } = req.user;
    
    const result = await this.totpService.enableMFA(userId, email);
    
    res.json({
      secret: result.secret,
      qrCode: result.qrCode,
      manualEntry: result.manualEntry
    });
  }
  
  async verifyMFASetup(req, res) {
    const userId = req.user.id;
    const { token } = req.body;
    
    const result = await this.totpService.verifyAndEnableMFA(userId, token);
    
    if (!result.success) {
      return res.status(400).json({ error: result.error });
    }
    
    res.json({
      success: true,
      backupCodes: result.backupCodes
    });
  }
  
  async disableMFA(req, res) {
    const userId = req.user.id;
    const { password } = req.body;
    
    try {
      await this.totpService.disableMFA(userId, password);
      res.json({ success: true });
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
  
  async regenerateBackupCodes(req, res) {
    const userId = req.user.id;
    const codes = await this.totpService.regenerateBackupCodes(userId);
    
    res.json({ backupCodes: codes });
  }
  
  async authenticateUser(email, password) {
    return { id: 'user-123', email, role: 'customer' };
  }
  
  async getUserById(userId) {
    return { id: userId, email: 'user@enbd.com', role: 'customer' };
  }
}

module.exports = { TOTPMFAService, SMSMFAService, MFAAuthenticationController };
```

**ENBD MFA Flow:**

```javascript
// 1. User logs in
POST /api/auth/login
{ "email": "customer@enbd.com", "password": "pass123" }

// Response (MFA required):
{
  "mfaRequired": true,
  "tempToken": "temp_token_here",
  "mfaMethods": ["totp", "sms"]
}

// 2. User enters TOTP code
POST /api/auth/mfa/verify
{
  "tempToken": "temp_token_here",
  "method": "totp",
  "code": "123456"
}

// Response (MFA verified):
{
  "accessToken": "jwt_token_here",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

---

### Q12. How do you implement API key authentication for third-party integrations?

**Answer:**

```javascript
const crypto = require('crypto');
const bcrypt = require('bcrypt');

/**
 * API Key Management Service
 */
class APIKeyService {
  constructor() {
    this.redis = new Redis();
  }
  
  /**
   * Generate API key for client
   */
  async generateAPIKey(clientData) {
    const keyId = `key_${crypto.randomBytes(8).toString('hex')}`;
    const secret = crypto.randomBytes(32).toString('base64url');
    const apiKey = `${keyId}.${secret}`;
    
    // Hash the secret for storage
    const hashedSecret = await bcrypt.hash(secret, 10);
    
    // Store key metadata
    await this.redis.hset(`apikey:${keyId}`, {
      keyId,
      hashedSecret,
      clientName: clientData.clientName,
      clientId: clientData.clientId,
      scope: JSON.stringify(clientData.scope || []),
      rateLimit: clientData.rateLimit || 1000,
      createdAt: new Date().toISOString(),
      lastUsed: null,
      enabled: 'true'
    });
    
    // Set expiration if specified
    if (clientData.expiresIn) {
      await this.redis.expire(`apikey:${keyId}`, clientData.expiresIn);
    }
    
    return {
      apiKey, // Return once - won't be retrievable again
      keyId,
      scope: clientData.scope,
      createdAt: new Date()
    };
  }
  
  /**
   * Verify API key
   */
  async verifyAPIKey(apiKey) {
    try {
      const [keyId, secret] = apiKey.split('.');
      
      if (!keyId || !secret) {
        return { valid: false, error: 'Invalid API key format' };
      }
      
      // Get key data
      const keyData = await this.redis.hgetall(`apikey:${keyId}`);
      
      if (!keyData || !keyData.hashedSecret) {
        return { valid: false, error: 'API key not found' };
      }
      
      // Check if enabled
      if (keyData.enabled !== 'true') {
        return { valid: false, error: 'API key disabled' };
      }
      
      // Verify secret
      const validSecret = await bcrypt.compare(secret, keyData.hashedSecret);
      
      if (!validSecret) {
        return { valid: false, error: 'Invalid API key' };
      }
      
      // Update last used
      await this.redis.hset(`apikey:${keyId}`, 'lastUsed', new Date().toISOString());
      
      return {
        valid: true,
        keyId,
        clientName: keyData.clientName,
        clientId: keyData.clientId,
        scope: JSON.parse(keyData.scope),
        rateLimit: parseInt(keyData.rateLimit)
      };
      
    } catch (error) {
      return { valid: false, error: error.message };
    }
  }
  
  /**
   * Revoke API key
   */
  async revokeAPIKey(keyId) {
    await this.redis.hset(`apikey:${keyId}`, 'enabled', 'false');
    
    // Also add to blacklist
    await this.redis.sadd('apikey:revoked', keyId);
  }
  
  /**
   * List API keys for client
   */
  async listAPIKeys(clientId) {
    const keys = await this.redis.keys('apikey:*');
    const result = [];
    
    for (const key of keys) {
      const data = await this.redis.hgetall(key);
      if (data.clientId === clientId) {
        result.push({
          keyId: data.keyId,
          clientName: data.clientName,
          scope: JSON.parse(data.scope),
          createdAt: data.createdAt,
          lastUsed: data.lastUsed,
          enabled: data.enabled === 'true'
        });
      }
    }
    
    return result;
  }
  
  /**
   * Rotate API key
   */
  async rotateAPIKey(oldKeyId, clientData) {
    // Generate new key
    const newKey = await this.generateAPIKey(clientData);
    
    // Mark old key for deprecation (grace period)
    await this.redis.hset(`apikey:${oldKeyId}`, 'deprecated', 'true');
    await this.redis.hset(`apikey:${oldKeyId}`, 'replacedBy', newKey.keyId);
    
    // Set expiration on old key (30 days grace period)
    await this.redis.expire(`apikey:${oldKeyId}`, 30 * 24 * 60 * 60);
    
    return newKey;
  }
}

/**
 * API Key Middleware
 */
class APIKeyMiddleware {
  constructor(apiKeyService) {
    this.apiKeyService = apiKeyService;
    this.rateLimiter = new RateLimiter();
  }
  
  authenticate() {
    return async (req, res, next) => {
      // Extract API key from header
      const apiKey = req.headers['x-api-key'] || req.headers['authorization']?.replace('Bearer ', '');
      
      if (!apiKey) {
        return res.status(401).json({
          error: 'API key required',
          code: 'MISSING_API_KEY'
        });
      }
      
      // Verify API key
      const result = await this.apiKeyService.verifyAPIKey(apiKey);
      
      if (!result.valid) {
        return res.status(401).json({
          error: result.error,
          code: 'INVALID_API_KEY'
        });
      }
      
      // Check rate limit
      const rateLimitOk = await this.rateLimiter.checkLimit(
        result.keyId,
        result.rateLimit
      );
      
      if (!rateLimitOk) {
        return res.status(429).json({
          error: 'Rate limit exceeded',
          code: 'RATE_LIMIT_EXCEEDED'
        });
      }
      
      // Attach client info to request
      req.client = {
        keyId: result.keyId,
        clientName: result.clientName,
        clientId: result.clientId,
        scope: result.scope
      };
      
      next();
    };
  }
  
  requireScope(...requiredScopes) {
    return (req, res, next) => {
      if (!req.client) {
        return res.status(401).json({ error: 'Not authenticated' });
      }
      
      const hasScope = requiredScopes.every(scope =>
        req.client.scope.includes(scope)
      );
      
      if (!hasScope) {
        return res.status(403).json({
          error: 'Insufficient scope',
          required: requiredScopes,
          current: req.client.scope
        });
      }
      
      next();
    };
  }
}

/**
 * Rate Limiter for API Keys
 */
class RateLimiter {
  constructor() {
    this.redis = new Redis();
  }
  
  async checkLimit(keyId, limit) {
    const key = `ratelimit:${keyId}:${this.getCurrentMinute()}`;
    
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      // First request in this window - set expiration
      await this.redis.expire(key, 60);
    }
    
    return current <= limit;
  }
  
  getCurrentMinute() {
    return Math.floor(Date.now() / 60000);
  }
  
  async getRemainingRequests(keyId, limit) {
    const key = `ratelimit:${keyId}:${this.getCurrentMinute()}`;
    const current = await this.redis.get(key) || 0;
    
    return Math.max(0, limit - parseInt(current));
  }
}

/**
 * Express Application with API Key Auth
 */
class APIKeyApplication {
  constructor() {
    this.app = express();
    this.apiKeyService = new APIKeyService();
    this.middleware = new APIKeyMiddleware(this.apiKeyService);
    
    this.setupRoutes();
  }
  
  setupRoutes() {
    // API key management endpoints
    app.post('/api/keys/generate', this.generateKey.bind(this));
    app.get('/api/keys', this.listKeys.bind(this));
    app.post('/api/keys/:keyId/revoke', this.revokeKey.bind(this));
    app.post('/api/keys/:keyId/rotate', this.rotateKey.bind(this));
    
    // Protected API endpoints
    app.get(
      '/api/accounts',
      this.middleware.authenticate(),
      this.middleware.requireScope('read:accounts'),
      this.getAccounts.bind(this)
    );
    
    app.post(
      '/api/transactions',
      this.middleware.authenticate(),
      this.middleware.requireScope('write:transactions'),
      this.createTransaction.bind(this)
    );
  }
  
  async generateKey(req, res) {
    // Requires admin authentication
    const { clientName, scope, rateLimit, expiresIn } = req.body;
    
    const apiKey = await this.apiKeyService.generateAPIKey({
      clientName,
      clientId: req.user.id,
      scope,
      rateLimit,
      expiresIn
    });
    
    res.json({
      apiKey: apiKey.apiKey, // Show only once!
      keyId: apiKey.keyId,
      scope: apiKey.scope,
      createdAt: apiKey.createdAt,
      message: 'Store this API key securely. It cannot be retrieved again.'
    });
  }
  
  async listKeys(req, res) {
    const keys = await this.apiKeyService.listAPIKeys(req.user.id);
    res.json({ keys });
  }
  
  async revokeKey(req, res) {
    const { keyId } = req.params;
    await this.apiKeyService.revokeAPIKey(keyId);
    res.json({ success: true });
  }
  
  async rotateKey(req, res) {
    const { keyId } = req.params;
    const { clientName, scope, rateLimit } = req.body;
    
    const newKey = await this.apiKeyService.rotateAPIKey(keyId, {
      clientName,
      clientId: req.user.id,
      scope,
      rateLimit
    });
    
    res.json(newKey);
  }
  
  async getAccounts(req, res) {
    // req.client contains API key info
    res.json({
      accounts: [],
      clientName: req.client.clientName
    });
  }
  
  async createTransaction(req, res) {
    res.json({ success: true });
  }
}

module.exports = { APIKeyService, APIKeyMiddleware, RateLimiter, APIKeyApplication };
```

**ENBD API Key Usage:**

```javascript
// 1. Generate API key (Admin)
POST /api/keys/generate
{
  "clientName": "Mobile App",
  "scope": ["read:accounts", "write:transactions"],
  "rateLimit": 1000,
  "expiresIn": 31536000
}

// Response:
{
  "apiKey": "key_abc123.very_long_secret_string",
  "keyId": "key_abc123",
  "scope": ["read:accounts", "write:transactions"]
}

// 2. Use API key
GET /api/accounts
Headers: {
  "X-API-Key": "key_abc123.very_long_secret_string"
}

// 3. Rate limit headers in response
Headers: {
  "X-RateLimit-Limit": "1000",
  "X-RateLimit-Remaining": "995",
  "X-RateLimit-Reset": "1699632060"
}
```

---

### Q13. How do you implement certificate-based authentication (mTLS)?

**Answer:**

```javascript
const fs = require('fs');
const https = require('https');
const express = require('express');
const forge = require('node-forge');

/**
 * mTLS (Mutual TLS) Authentication Service
 */
class MTLSService {
  constructor() {
    this.trustedCAs = new Map();
    this.clientCertificates = new Map();
    this.loadTrustedCAs();
  }
  
  /**
   * Load trusted Certificate Authorities
   */
  loadTrustedCAs() {
    const caCert = fs.readFileSync('./certs/ca-cert.pem', 'utf8');
    this.trustedCAs.set('enbd-ca', forge.pki.certificateFromPem(caCert));
  }
  
  /**
   * Create HTTPS server with mTLS
   */
  createMTLSServer(app) {
    const serverOptions = {
      key: fs.readFileSync('./certs/server-key.pem'),
      cert: fs.readFileSync('./certs/server-cert.pem'),
      ca: fs.readFileSync('./certs/ca-cert.pem'),
      requestCert: true,              // Request client certificate
      rejectUnauthorized: true,       // Reject unauthorized clients
      minVersion: 'TLSv1.2'          // Minimum TLS version
    };
    
    const server = https.createServer(serverOptions, app);
    
    return server;
  }
  
  /**
   * Verify client certificate middleware
   */
  verifyClientCertificate() {
    return (req, res, next) => {
      const cert = req.socket.getPeerCertificate();
      
      if (!cert || !cert.subject) {
        return res.status(401).json({
          error: 'Client certificate required'
        });
      }
      
      // Verify certificate details
      const validation = this.validateCertificate(cert);
      
      if (!validation.valid) {
        return res.status(403).json({
          error: 'Invalid client certificate',
          details: validation.error
        });
      }
      
      // Extract client info from certificate
      req.client = {
        commonName: cert.subject.CN,
        organization: cert.subject.O,
        organizationalUnit: cert.subject.OU,
        serialNumber: cert.serialNumber,
        fingerprint: cert.fingerprint,
        validFrom: cert.valid_from,
        validTo: cert.valid_to
      };
      
      next();
    };
  }
  
  /**
   * Validate certificate
   */
  validateCertificate(cert) {
    // 1. Check if certificate is expired
    const now = new Date();
    const validFrom = new Date(cert.valid_from);
    const validTo = new Date(cert.valid_to);
    
    if (now < validFrom || now > validTo) {
      return { valid: false, error: 'Certificate expired or not yet valid' };
    }
    
    // 2. Check certificate revocation list (CRL)
    if (this.isRevoked(cert.serialNumber)) {
      return { valid: false, error: 'Certificate has been revoked' };
    }
    
    // 3. Verify issuer
    if (!cert.issuer || cert.issuer.CN !== 'ENBD Certificate Authority') {
      return { valid: false, error: 'Invalid certificate issuer' };
    }
    
    // 4. Check key usage
    if (!this.hasValidKeyUsage(cert)) {
      return { valid: false, error: 'Invalid key usage' };
    }
    
    return { valid: true };
  }
  
  /**
   * Check if certificate is revoked
   */
  isRevoked(serialNumber) {
    // Check against Certificate Revocation List (CRL)
    // In production, check against OCSP or CRL
    const revokedCerts = new Set(['123456', '789012']);
    return revokedCerts.has(serialNumber);
  }
  
  hasValidKeyUsage(cert) {
    // Verify certificate has correct key usage extensions
    return true; // Simplified
  }
  
  /**
   * Generate client certificate (for testing)
   */
  generateClientCertificate(clientData) {
    const pki = forge.pki;
    
    // Generate key pair
    const keys = pki.rsa.generateKeyPair(2048);
    
    // Create certificate
    const cert = pki.createCertificate();
    cert.publicKey = keys.publicKey;
    cert.serialNumber = '01';
    cert.validity.notBefore = new Date();
    cert.validity.notAfter = new Date();
    cert.validity.notAfter.setFullYear(cert.validity.notBefore.getFullYear() + 1);
    
    const attrs = [
      { name: 'commonName', value: clientData.commonName },
      { name: 'countryName', value: 'AE' },
      { name: 'stateOrProvinceName', value: 'Dubai' },
      { name: 'organizationName', value: clientData.organization },
      { name: 'organizationalUnitName', value: clientData.unit }
    ];
    
    cert.setSubject(attrs);
    cert.setIssuer(attrs); // Self-signed for testing
    
    // Add extensions
    cert.setExtensions([
      {
        name: 'basicConstraints',
        cA: false
      },
      {
        name: 'keyUsage',
        digitalSignature: true,
        keyEncipherment: true
      },
      {
        name: 'extKeyUsage',
        clientAuth: true
      }
    ]);
    
    // Sign certificate
    cert.sign(keys.privateKey, forge.md.sha256.create());
    
    // Convert to PEM format
    const pemCert = pki.certificateToPem(cert);
    const pemKey = pki.privateKeyToPem(keys.privateKey);
    
    return {
      certificate: pemCert,
      privateKey: pemKey
    };
  }
}

/**
 * Express Application with mTLS
 */
class MTLSBankingAPI {
  constructor() {
    this.app = express();
    this.mtlsService = new MTLSService();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    
    // Verify client certificate on all routes
    this.app.use(this.mtlsService.verifyClientCertificate());
    
    // Log client certificate info
    this.app.use((req, res, next) => {
      console.log('Client authenticated:', req.client.commonName);
      next();
    });
  }
  
  setupRoutes() {
    app.get('/api/accounts', this.getAccounts.bind(this));
    app.post('/api/transactions', this.createTransaction.bind(this));
    
    // Health check (no cert required)
    app.get('/health', (req, res) => {
      res.json({ status: 'healthy' });
    });
  }
  
  async getAccounts(req, res) {
    // req.client contains certificate info
    res.json({
      accounts: [],
      authenticatedAs: req.client.commonName,
      organization: req.client.organization
    });
  }
  
  async createTransaction(req, res) {
    // Audit log with certificate details
    console.log('Transaction by:', req.client.commonName);
    
    res.json({ success: true });
  }
  
  start(port = 3443) {
    const server = this.mtlsService.createMTLSServer(this.app);
    
    server.listen(port, () => {
      console.log(`mTLS server running on https://localhost:${port}`);
    });
  }
}

/**
 * mTLS Client
 */
class MTLSClient {
  constructor(clientCertPath, clientKeyPath, caPath) {
    this.httpsAgent = new https.Agent({
      cert: fs.readFileSync(clientCertPath),
      key: fs.readFileSync(clientKeyPath),
      ca: fs.readFileSync(caPath),
      rejectUnauthorized: true
    });
  }
  
  async request(url, options = {}) {
    const axios = require('axios');
    
    return axios({
      url,
      ...options,
      httpsAgent: this.httpsAgent
    });
  }
}

module.exports = { MTLSService, MTLSBankingAPI, MTLSClient };
```

**ENBD mTLS Usage:**

```bash
# Generate certificates
openssl genrsa -out ca-key.pem 2048
openssl req -new -x509 -key ca-key.pem -out ca-cert.pem -days 365

# Client certificate
openssl genrsa -out client-key.pem 2048
openssl req -new -key client-key.pem -out client-csr.pem
openssl x509 -req -in client-csr.pem -CA ca-cert.pem -CAkey ca-key.pem -out client-cert.pem

# Use client certificate
curl --cert client-cert.pem --key client-key.pem --cacert ca-cert.pem https://api.enbd.com/accounts
```

---

### Q14. How do you implement session management and CSRF protection?

**Answer:**

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const crypto = require('crypto');

/**
 * Session Management Service
 */
class SessionManagementService {
  constructor() {
    this.redis = new Redis();
    this.sessionStore = new RedisStore({
      client: this.redis,
      prefix: 'sess:',
      ttl: 24 * 60 * 60 // 24 hours
    });
  }
  
  /**
   * Configure session middleware
   */
  getSessionMiddleware() {
    return session({
      store: this.sessionStore,
      secret: process.env.SESSION_SECRET,
      name: 'enbd.sid',
      resave: false,
      saveUninitialized: false,
      cookie: {
        secure: true,           // HTTPS only
        httpOnly: true,         // Not accessible via JavaScript
        sameSite: 'strict',     // CSRF protection
        maxAge: 24 * 60 * 60 * 1000, // 24 hours
        domain: '.enbd.com'
      },
      rolling: true,            // Reset expiration on each request
      unset: 'destroy'
    });
  }
  
  /**
   * Create session for user
   */
  async createSession(userId, userData, req) {
    return new Promise((resolve, reject) => {
      req.session.userId = userId;
      req.session.userData = userData;
      req.session.createdAt = Date.now();
      req.session.lastActivity = Date.now();
      req.session.ipAddress = req.ip;
      req.session.userAgent = req.headers['user-agent'];
      
      req.session.save((err) => {
        if (err) reject(err);
        else resolve(req.session.id);
      });
    });
  }
  
  /**
   * Destroy session
   */
  async destroySession(req) {
    return new Promise((resolve, reject) => {
      req.session.destroy((err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }
  
  /**
   * Regenerate session ID (after login)
   */
  async regenerateSession(req) {
    const oldData = { ...req.session };
    
    return new Promise((resolve, reject) => {
      req.session.regenerate((err) => {
        if (err) reject(err);
        else {
          // Restore session data
          Object.assign(req.session, oldData);
          resolve(req.session.id);
        }
      });
    });
  }
  
  /**
   * Get all active sessions for user
   */
  async getUserSessions(userId) {
    const keys = await this.redis.keys('sess:*');
    const sessions = [];
    
    for (const key of keys) {
      const data = await this.redis.get(key);
      if (data) {
        const session = JSON.parse(data);
        if (session.userId === userId) {
          sessions.push({
            sessionId: key.replace('sess:', ''),
            createdAt: session.createdAt,
            lastActivity: session.lastActivity,
            ipAddress: session.ipAddress,
            userAgent: session.userAgent
          });
        }
      }
    }
    
    return sessions;
  }
  
  /**
   * Destroy all sessions for user (except current)
   */
  async destroyOtherSessions(userId, currentSessionId) {
    const keys = await this.redis.keys('sess:*');
    let destroyedCount = 0;
    
    for (const key of keys) {
      const sessionId = key.replace('sess:', '');
      
      if (sessionId === currentSessionId) continue;
      
      const data = await this.redis.get(key);
      if (data) {
        const session = JSON.parse(data);
        if (session.userId === userId) {
          await this.redis.del(key);
          destroyedCount++;
        }
      }
    }
    
    return destroyedCount;
  }
}

/**
 * CSRF Protection Service
 */
class CSRFProtectionService {
  /**
   * Generate CSRF token
   */
  generateToken(req) {
    const token = crypto.randomBytes(32).toString('hex');
    
    // Store in session
    req.session.csrfToken = token;
    
    return token;
  }
  
  /**
   * Verify CSRF token
   */
  verifyToken(req, token) {
    if (!req.session || !req.session.csrfToken) {
      return false;
    }
    
    // Constant-time comparison to prevent timing attacks
    const expected = Buffer.from(req.session.csrfToken);
    const actual = Buffer.from(token);
    
    if (expected.length !== actual.length) {
      return false;
    }
    
    return crypto.timingSafeEqual(expected, actual);
  }
  
  /**
   * CSRF middleware
   */
  middleware() {
    return (req, res, next) => {
      // Skip for safe methods
      if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        return next();
      }
      
      // Extract token from header or body
      const token = req.headers['x-csrf-token'] || 
                   req.body._csrf || 
                   req.query._csrf;
      
      if (!token) {
        return res.status(403).json({
          error: 'CSRF token required'
        });
      }
      
      // Verify token
      if (!this.verifyToken(req, token)) {
        return res.status(403).json({
          error: 'Invalid CSRF token'
        });
      }
      
      next();
    };
  }
  
  /**
   * Get CSRF token endpoint
   */
  getTokenEndpoint() {
    return (req, res) => {
      const token = this.generateToken(req);
      res.json({ csrfToken: token });
    };
  }
}

/**
 * Complete Secure Session Application
 */
class SecureSessionApplication {
  constructor() {
    this.app = express();
    this.sessionService = new SessionManagementService();
    this.csrfService = new CSRFProtectionService();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
    
    // Session middleware
    this.app.use(this.sessionService.getSessionMiddleware());
    
    // CSRF protection (after session)
    this.app.use(this.csrfService.middleware());
    
    // Activity tracking
    this.app.use((req, res, next) => {
      if (req.session && req.session.userId) {
        req.session.lastActivity = Date.now();
      }
      next();
    });
  }
  
  setupRoutes() {
    // Get CSRF token
    app.get('/api/csrf-token', this.csrfService.getTokenEndpoint());
    
    // Login
    app.post('/api/auth/login', async (req, res) => {
      const { email, password } = req.body;
      
      const user = await this.authenticateUser(email, password);
      
      if (!user) {
        return res.status(401).json({ error: 'Invalid credentials' });
      }
      
      // Regenerate session ID to prevent session fixation
      await this.sessionService.regenerateSession(req);
      
      // Create session
      await this.sessionService.createSession(user.id, {
        email: user.email,
        role: user.role
      }, req);
      
      // Generate CSRF token
      const csrfToken = this.csrfService.generateToken(req);
      
      res.json({
        success: true,
        csrfToken,
        user: {
          id: user.id,
          email: user.email,
          role: user.role
        }
      });
    });
    
    // Logout
    app.post('/api/auth/logout', async (req, res) => {
      await this.sessionService.destroySession(req);
      res.json({ success: true });
    });
    
    // Get active sessions
    app.get('/api/user/sessions', async (req, res) => {
      if (!req.session.userId) {
        return res.status(401).json({ error: 'Not authenticated' });
      }
      
      const sessions = await this.sessionService.getUserSessions(req.session.userId);
      res.json({ sessions });
    });
    
    // Logout other sessions
    app.post('/api/user/logout-others', async (req, res) => {
      if (!req.session.userId) {
        return res.status(401).json({ error: 'Not authenticated' });
      }
      
      const count = await this.sessionService.destroyOtherSessions(
        req.session.userId,
        req.session.id
      );
      
      res.json({ success: true, destroyedCount: count });
    });
    
    // Protected route
    app.post('/api/transactions', async (req, res) => {
      if (!req.session.userId) {
        return res.status(401).json({ error: 'Not authenticated' });
      }
      
      res.json({ success: true });
    });
  }
  
  async authenticateUser(email, password) {
    return { id: 'user-123', email, role: 'customer' };
  }
}

module.exports = { SessionManagementService, CSRFProtectionService, SecureSessionApplication };
```

**ENBD Session & CSRF Flow:**

```javascript
// 1. Get CSRF token
GET /api/csrf-token
Response: { "csrfToken": "abc123..." }

// 2. Login with CSRF token
POST /api/auth/login
Headers: { "X-CSRF-Token": "abc123..." }
Body: { "email": "user@enbd.com", "password": "pass" }

// 3. Make protected request
POST /api/transactions
Headers: {
  "X-CSRF-Token": "abc123...",
  "Cookie": "enbd.sid=session_id_here"
}
```

---

### Q15. How do you implement Single Sign-On (SSO) with SAML?

**Answer:**

```javascript
const saml = require('passport-saml');
const passport = require('passport');
const xml2js = require('xml2js');
const zlib = require('zlib');

/**
 * SAML SSO Service for ENBD
 */
class SAMLSSOService {
  constructor() {
    this.setupSAMLStrategy();
  }
  
  /**
   * Configure SAML strategy
   */
  setupSAMLStrategy() {
    const strategy = new saml.Strategy(
      {
        // Service Provider (SP) - Your app
        callbackUrl: 'https://app.enbd.com/auth/saml/callback',
        entryPoint: 'https://idp.enbd.com/saml/sso',
        issuer: 'enbd-banking-app',
        
        // Identity Provider (IdP) certificate
        cert: fs.readFileSync('./certs/idp-cert.pem', 'utf8'),
        
        // Private key for signing requests
        privateKey: fs.readFileSync('./certs/sp-key.pem', 'utf8'),
        
        // Signature algorithm
        signatureAlgorithm: 'sha256',
        
        // Additional options
        identifierFormat: 'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent',
        wantAssertionsSigned: true,
        wantAuthnResponseSigned: true,
        
        // Attribute mapping
        attributeMap: {
          'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress': 'email',
          'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name': 'name',
          'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/role': 'role'
        }
      },
      (profile, done) => {
        // Verify and create/update user
        this.verifyUser(profile)
          .then(user => done(null, user))
          .catch(err => done(err));
      }
    );
    
    passport.use('saml', strategy);
  }
  
  /**
   * Generate SAML Authentication Request
   */
  generateAuthRequest() {
    const requestId = '_' + crypto.randomBytes(21).toString('hex');
    const issueInstant = new Date().toISOString();
    
    const request = `
      <samlp:AuthnRequest 
        xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
        xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
        ID="${requestId}"
        Version="2.0"
        IssueInstant="${issueInstant}"
        Destination="https://idp.enbd.com/saml/sso"
        ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
        AssertionConsumerServiceURL="https://app.enbd.com/auth/saml/callback">
        
        <saml:Issuer>enbd-banking-app</saml:Issuer>
        
        <samlp:NameIDPolicy
          Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent"
          AllowCreate="true"/>
          
        <samlp:RequestedAuthnContext Comparison="exact">
          <saml:AuthnContextClassRef>
            urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
          </saml:AuthnContextClassRef>
        </samlp:RequestedAuthnContext>
      </samlp:AuthnRequest>
    `;
    
    return request;
  }
  
  /**
   * Parse SAML Response
   */
  async parseSAMLResponse(samlResponse) {
    const parser = new xml2js.Parser({ explicitArray: false });
    
    // Decode base64
    const decoded = Buffer.from(samlResponse, 'base64').toString('utf8');
    
    // Parse XML
    const result = await parser.parseStringPromise(decoded);
    
    // Extract assertion
    const assertion = result['saml2p:Response']['saml2:Assertion'];
    
    return {
      nameID: assertion['saml2:Subject']['saml2:NameID']._,
      attributes: this.extractAttributes(assertion),
      conditions: assertion['saml2:Conditions']
    };
  }
  
  extractAttributes(assertion) {
    const attributeStatement = assertion['saml2:AttributeStatement'];
    const attributes = {};
    
    if (attributeStatement && attributeStatement['saml2:Attribute']) {
      const attrs = Array.isArray(attributeStatement['saml2:Attribute'])
        ? attributeStatement['saml2:Attribute']
        : [attributeStatement['saml2:Attribute']];
      
      for (const attr of attrs) {
        const name = attr.$.Name;
        const value = attr['saml2:AttributeValue'];
        attributes[name] = value;
      }
    }
    
    return attributes;
  }
  
  /**
   * Verify user from SAML profile
   */
  async verifyUser(profile) {
    // Find or create user in database
    let user = await db.users.findOne({ samlNameId: profile.nameID });
    
    if (!user) {
      user = await db.users.create({
        samlNameId: profile.nameID,
        email: profile.email,
        name: profile.name,
        role: profile.role || 'customer',
        authProvider: 'saml-sso'
      });
    } else {
      // Update user info
      await db.users.update(user.id, {
        email: profile.email,
        name: profile.name,
        lastLogin: new Date()
      });
    }
    
    return user;
  }
  
  /**
   * Generate SAML Metadata (for IdP configuration)
   */
  generateMetadata() {
    const cert = fs.readFileSync('./certs/sp-cert.pem', 'utf8')
      .replace(/-----BEGIN CERTIFICATE-----/, '')
      .replace(/-----END CERTIFICATE-----/, '')
      .replace(/\n/g, '');
    
    return `
      <md:EntityDescriptor 
        xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
        entityID="enbd-banking-app">
        
        <md:SPSSODescriptor 
          AuthnRequestsSigned="true"
          WantAssertionsSigned="true"
          protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
          
          <md:KeyDescriptor use="signing">
            <ds:KeyInfo xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
              <ds:X509Data>
                <ds:X509Certificate>${cert}</ds:X509Certificate>
              </ds:X509Data>
            </ds:KeyInfo>
          </md:KeyDescriptor>
          
          <md:NameIDFormat>
            urn:oasis:names:tc:SAML:2.0:nameid-format:persistent
          </md:NameIDFormat>
          
          <md:AssertionConsumerService 
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
            Location="https://app.enbd.com/auth/saml/callback"
            index="1"/>
        </md:SPSSODescriptor>
      </md:EntityDescriptor>
    `;
  }
}

/**
 * Express Application with SAML SSO
 */
class SAMLSSOApplication {
  constructor() {
    this.app = express();
    this.samlService = new SAMLSSOService();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
    this.app.use(session({ secret: 'secret', resave: false, saveUninitialized: false }));
    this.app.use(passport.initialize());
    this.app.use(passport.session());
    
    // Serialize user
    passport.serializeUser((user, done) => {
      done(null, user.id);
    });
    
    passport.deserializeUser(async (id, done) => {
      const user = await db.users.findById(id);
      done(null, user);
    });
  }
  
  setupRoutes() {
    // SAML metadata endpoint
    this.app.get('/auth/saml/metadata', (req, res) => {
      res.header('Content-Type', 'application/xml');
      res.send(this.samlService.generateMetadata());
    });
    
    // Initiate SAML login
    this.app.get('/auth/saml/login',
      passport.authenticate('saml', {
        failureRedirect: '/login',
        failureFlash: true
      })
    );
    
    // SAML callback (assertion consumer service)
    this.app.post('/auth/saml/callback',
      passport.authenticate('saml', {
        failureRedirect: '/login',
        failureFlash: true
      }),
      (req, res) => {
        // Successful authentication
        res.redirect('/dashboard');
      }
    );
    
    // Logout
    this.app.get('/auth/saml/logout', (req, res) => {
      req.logout(() => {
        // Generate SAML logout request
        const logoutRequest = this.generateLogoutRequest(req.user);
        res.redirect(logoutRequest);
      });
    });
    
    // Protected route
    this.app.get('/dashboard', this.ensureAuthenticated, (req, res) => {
      res.json({
        user: req.user,
        message: 'Welcome to ENBD Dashboard'
      });
    });
  }
  
  ensureAuthenticated(req, res, next) {
    if (req.isAuthenticated()) {
      return next();
    }
    res.redirect('/auth/saml/login');
  }
  
  generateLogoutRequest(user) {
    // Generate SAML logout request
    const requestId = '_' + crypto.randomBytes(21).toString('hex');
    const logoutUrl = `https://idp.enbd.com/saml/logout?SAMLRequest=${encodeURIComponent(requestId)}`;
    return logoutUrl;
  }
}

module.exports = { SAMLSSOService, SAMLSSOApplication };
```

**ENBD SAML SSO Flow:**

```
1. User → SP: Access protected resource
2. SP → User: Redirect to IdP with SAML request
3. User → IdP: Authenticate with credentials
4. IdP → User: Redirect back to SP with SAML response
5. User → SP: POST SAML response to callback URL
6. SP: Verify SAML response and create session
7. SP → User: Grant access to protected resource
```

---

**Summary Q11-Q15:**
- Multi-Factor Authentication (TOTP & SMS) ✅
- API key authentication for third-party ✅
- Certificate-based authentication (mTLS) ✅
- Session management & CSRF protection ✅
- Single Sign-On (SSO) with SAML ✅

Continuing with remaining security questions...
