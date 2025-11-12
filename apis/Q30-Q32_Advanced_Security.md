# APIs Interview Questions (Q30-Q32): Advanced Security

## Q30: How do you protect banking APIs against OWASP API Security Top 10 vulnerabilities?

**Answer:**

The OWASP API Security Top 10 identifies critical security risks for APIs. Banking systems must implement comprehensive protections against these threats.

### OWASP API Security Top 10 Protection:

```javascript
// owasp-protection.js
const express = require('express');
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');
const validator = require('validator');
const jwt = require('jsonwebtoken');

/**
 * API1:2023 - Broken Object Level Authorization (BOLA)
 * Attackers can access objects they shouldn't by manipulating IDs
 */
class BOLAProtection {
    /**
     * Middleware to verify resource ownership
     */
    static verifyOwnership(resourceType) {
        return async (req, res, next) => {
            try {
                const userId = req.user.id;
                const resourceId = req.params.id;
                
                // Check if user owns the resource
                const owns = await this.checkOwnership(userId, resourceId, resourceType);
                
                if (!owns && !req.user.roles.includes('admin')) {
                    return res.status(403).json({
                        error: 'Forbidden',
                        message: 'You do not have access to this resource'
                    });
                }
                
                next();
            } catch (error) {
                next(error);
            }
        };
    }

    static async checkOwnership(userId, resourceId, resourceType) {
        // Example: Check account ownership
        if (resourceType === 'account') {
            const result = await db.query(
                'SELECT 1 FROM accounts WHERE id = $1 AND customer_id = $2',
                [resourceId, userId]
            );
            return result.rows.length > 0;
        }
        
        // Example: Check transaction ownership
        if (resourceType === 'transaction') {
            const result = await db.query(`
                SELECT 1 FROM transactions t
                JOIN accounts a ON t.account_id = a.id
                WHERE t.id = $1 AND a.customer_id = $2
            `, [resourceId, userId]);
            return result.rows.length > 0;
        }
        
        return false;
    }
}

/**
 * API2:2023 - Broken Authentication
 * Weak authentication mechanisms
 */
class AuthenticationProtection {
    /**
     * Strong password validation
     */
    static validatePassword(password) {
        const errors = [];
        
        if (password.length < 12) {
            errors.push('Password must be at least 12 characters');
        }
        
        if (!/[A-Z]/.test(password)) {
            errors.push('Password must contain uppercase letters');
        }
        
        if (!/[a-z]/.test(password)) {
            errors.push('Password must contain lowercase letters');
        }
        
        if (!/[0-9]/.test(password)) {
            errors.push('Password must contain numbers');
        }
        
        if (!/[!@#$%^&*]/.test(password)) {
            errors.push('Password must contain special characters');
        }
        
        // Check against common passwords
        const commonPasswords = ['Password123!', 'Welcome123!', 'Admin123!'];
        if (commonPasswords.includes(password)) {
            errors.push('Password is too common');
        }
        
        return {
            valid: errors.length === 0,
            errors
        };
    }

    /**
     * Multi-factor authentication
     */
    static async verifyMFA(userId, mfaToken) {
        const speakeasy = require('speakeasy');
        
        // Get user's MFA secret from database
        const result = await db.query(
            'SELECT mfa_secret FROM customers WHERE id = $1',
            [userId]
        );
        
        if (!result.rows[0]?.mfa_secret) {
            throw new Error('MFA not configured');
        }
        
        const verified = speakeasy.totp.verify({
            secret: result.rows[0].mfa_secret,
            encoding: 'base32',
            token: mfaToken,
            window: 2  // Allow 2 time steps before/after
        });
        
        return verified;
    }

    /**
     * JWT with short expiry and refresh tokens
     */
    static generateTokens(userId) {
        const accessToken = jwt.sign(
            { userId, type: 'access' },
            process.env.JWT_SECRET,
            { expiresIn: '15m' }  // Short-lived
        );
        
        const refreshToken = jwt.sign(
            { userId, type: 'refresh' },
            process.env.JWT_REFRESH_SECRET,
            { expiresIn: '7d' }
        );
        
        return { accessToken, refreshToken };
    }

    /**
     * Account lockout after failed attempts
     */
    static async handleFailedLogin(email) {
        const key = `login_attempts:${email}`;
        const attempts = await redis.incr(key);
        
        if (attempts === 1) {
            await redis.expire(key, 900); // 15 minutes
        }
        
        if (attempts >= 5) {
            await redis.setex(`account_locked:${email}`, 1800, '1'); // Lock for 30 min
            throw new Error('Account locked due to too many failed attempts');
        }
        
        return { attemptsRemaining: 5 - attempts };
    }
}

/**
 * API3:2023 - Broken Object Property Level Authorization
 * Exposing sensitive data in API responses
 */
class DataExposureProtection {
    /**
     * Sanitize response data based on user role
     */
    static sanitizeResponse(data, userRole) {
        const sensitiveFields = {
            customer: ['ssn', 'password', 'mfa_secret', 'internal_notes'],
            account: ['internal_id', 'risk_score', 'fraud_flags'],
            transaction: ['internal_reference', 'processor_response']
        };
        
        const allowedRoles = {
            customer: ['admin', 'support'],
            sensitive: ['admin']
        };
        
        // Remove sensitive fields based on role
        const sanitized = { ...data };
        
        Object.keys(sensitiveFields).forEach(entityType => {
            sensitiveFields[entityType].forEach(field => {
                if (sanitized[field] !== undefined) {
                    if (!allowedRoles.sensitive.includes(userRole)) {
                        delete sanitized[field];
                    }
                }
            });
        });
        
        return sanitized;
    }

    /**
     * Mask sensitive data
     */
    static maskSensitiveData(data, fields) {
        const masked = { ...data };
        
        fields.forEach(field => {
            if (masked[field]) {
                if (field === 'account_number') {
                    // Mask: AE070331234567890123456 -> AE07***********3456
                    masked[field] = masked[field].replace(/(.{4})(.*)(.{4})/, '$1***********$3');
                } else if (field === 'phone') {
                    // Mask: +971501234567 -> +971*****4567
                    masked[field] = masked[field].replace(/(.{4})(.*)(.{4})/, '$1*****$3');
                } else if (field === 'email') {
                    // Mask: john.doe@email.com -> j***@email.com
                    masked[field] = masked[field].replace(/^(.)(.*)(@.*)$/, '$1***$3');
                }
            }
        });
        
        return masked;
    }
}

/**
 * API4:2023 - Unrestricted Resource Consumption
 * Rate limiting and resource quotas
 */
class ResourceProtection {
    /**
     * Advanced rate limiting with different tiers
     */
    static createRateLimiter(tier = 'free') {
        const limits = {
            free: { windowMs: 60000, max: 10 },
            basic: { windowMs: 60000, max: 100 },
            premium: { windowMs: 60000, max: 1000 },
            enterprise: { windowMs: 60000, max: 10000 }
        };
        
        const config = limits[tier] || limits.free;
        
        return rateLimit({
            windowMs: config.windowMs,
            max: config.max,
            message: {
                error: 'TooManyRequests',
                message: `Rate limit exceeded. Limit: ${config.max} requests per minute`
            },
            standardHeaders: true,
            legacyHeaders: false,
            handler: (req, res) => {
                res.status(429).json({
                    error: 'TooManyRequests',
                    message: `Rate limit exceeded`,
                    retryAfter: res.getHeader('Retry-After')
                });
            }
        });
    }

    /**
     * Request size limits
     */
    static requestSizeLimits() {
        return express.json({
            limit: '1mb',
            verify: (req, res, buf, encoding) => {
                if (buf.length > 1048576) { // 1MB
                    throw new Error('Request entity too large');
                }
            }
        });
    }

    /**
     * Query complexity limits for GraphQL/complex queries
     */
    static async validateQueryComplexity(query) {
        const maxDepth = 5;
        const maxNodes = 100;
        
        const depth = this.calculateQueryDepth(query);
        const nodes = this.countQueryNodes(query);
        
        if (depth > maxDepth) {
            throw new Error(`Query depth ${depth} exceeds maximum ${maxDepth}`);
        }
        
        if (nodes > maxNodes) {
            throw new Error(`Query nodes ${nodes} exceeds maximum ${maxNodes}`);
        }
    }

    static calculateQueryDepth(query, depth = 0) {
        // Simplified depth calculation
        return depth;
    }

    static countQueryNodes(query) {
        // Simplified node counting
        return 0;
    }
}

/**
 * API5:2023 - Broken Function Level Authorization
 * Ensure users can only access functions they're authorized for
 */
class FunctionAuthorizationProtection {
    /**
     * Role-based access control middleware
     */
    static requireRole(...allowedRoles) {
        return (req, res, next) => {
            if (!req.user) {
                return res.status(401).json({
                    error: 'Unauthorized',
                    message: 'Authentication required'
                });
            }
            
            const hasRole = allowedRoles.some(role => 
                req.user.roles.includes(role)
            );
            
            if (!hasRole) {
                return res.status(403).json({
                    error: 'Forbidden',
                    message: `Requires one of: ${allowedRoles.join(', ')}`
                });
            }
            
            next();
        };
    }

    /**
     * Permission-based access control
     */
    static requirePermission(...requiredPermissions) {
        return (req, res, next) => {
            if (!req.user?.permissions) {
                return res.status(403).json({
                    error: 'Forbidden',
                    message: 'Insufficient permissions'
                });
            }
            
            const hasAllPermissions = requiredPermissions.every(perm =>
                req.user.permissions.includes(perm)
            );
            
            if (!hasAllPermissions) {
                return res.status(403).json({
                    error: 'Forbidden',
                    message: `Requires permissions: ${requiredPermissions.join(', ')}`
                });
            }
            
            next();
        };
    }

    /**
     * Example: Admin-only endpoint
     */
    static setupAdminRoutes(app) {
        // ❌ BAD: No authorization check
        // app.delete('/api/users/:id', deleteUser);
        
        // ✅ GOOD: Require admin role
        app.delete('/api/users/:id',
            FunctionAuthorizationProtection.requireRole('admin'),
            async (req, res) => {
                // Delete user logic
            }
        );
        
        // ✅ GOOD: Require specific permission
        app.post('/api/accounts/:id/freeze',
            FunctionAuthorizationProtection.requirePermission('accounts:freeze'),
            async (req, res) => {
                // Freeze account logic
            }
        );
    }
}

/**
 * API6:2023 - Unrestricted Access to Sensitive Business Flows
 * Protect critical operations with additional verification
 */
class BusinessFlowProtection {
    /**
     * Transaction signing for critical operations
     */
    static async verifyTransactionSignature(req, res, next) {
        const { amount, recipientAccount, signature } = req.body;
        
        // For large transactions, require additional verification
        if (amount > 50000) {
            if (!signature) {
                return res.status(400).json({
                    error: 'SignatureRequired',
                    message: 'Large transactions require digital signature'
                });
            }
            
            const valid = await this.verifyDigitalSignature(
                req.user.id,
                { amount, recipientAccount },
                signature
            );
            
            if (!valid) {
                return res.status(403).json({
                    error: 'InvalidSignature',
                    message: 'Transaction signature verification failed'
                });
            }
        }
        
        next();
    }

    static async verifyDigitalSignature(userId, data, signature) {
        // Implement digital signature verification
        return true;
    }

    /**
     * Velocity checks for suspicious activity
     */
    static async checkTransactionVelocity(userId, amount) {
        const key = `velocity:${userId}`;
        
        // Check transaction count in last hour
        const hourKey = `${key}:hour`;
        const hourCount = await redis.incr(hourKey);
        if (hourCount === 1) {
            await redis.expire(hourKey, 3600);
        }
        
        if (hourCount > 10) {
            throw new Error('Too many transactions in the last hour');
        }
        
        // Check total amount in last day
        const dayKey = `${key}:day:amount`;
        const dayAmount = await redis.incrby(dayKey, amount);
        if (dayAmount - amount === 0) {
            await redis.expire(dayKey, 86400);
        }
        
        if (dayAmount > 100000) {
            throw new Error('Daily transaction limit exceeded');
        }
        
        return true;
    }

    /**
     * Anomaly detection
     */
    static async detectAnomalies(transaction) {
        const { userId, amount, location, deviceId } = transaction;
        
        // Check if transaction is unusual for this user
        const avgTransaction = await this.getUserAverageTransaction(userId);
        
        if (amount > avgTransaction * 5) {
            // Transaction is 5x larger than usual
            await this.flagForReview(transaction, 'unusual_amount');
        }
        
        // Check location
        const lastLocation = await this.getUserLastLocation(userId);
        if (lastLocation && this.calculateDistance(location, lastLocation) > 1000) {
            // User is 1000km away from last transaction
            await this.flagForReview(transaction, 'unusual_location');
        }
        
        // Check device
        const knownDevice = await this.isKnownDevice(userId, deviceId);
        if (!knownDevice) {
            await this.flagForReview(transaction, 'unknown_device');
        }
    }

    static async getUserAverageTransaction(userId) {
        const result = await db.query(`
            SELECT AVG(amount) as avg_amount
            FROM transactions
            WHERE user_id = $1
            AND created_at > NOW() - INTERVAL '30 days'
        `, [userId]);
        
        return parseFloat(result.rows[0]?.avg_amount || 0);
    }

    static async flagForReview(transaction, reason) {
        await db.query(`
            INSERT INTO fraud_reviews (transaction_id, reason, created_at)
            VALUES ($1, $2, NOW())
        `, [transaction.id, reason]);
        
        // Send notification to fraud team
        console.log(`⚠️ Transaction ${transaction.id} flagged for review: ${reason}`);
    }
}

/**
 * API7:2023 - Server Side Request Forgery (SSRF)
 * Validate and sanitize URLs
 */
class SSRFProtection {
    /**
     * Validate external URLs
     */
    static validateURL(url) {
        try {
            const parsed = new URL(url);
            
            // Block private IP ranges
            const blockedHosts = [
                'localhost',
                '127.0.0.1',
                '0.0.0.0',
                '::1'
            ];
            
            if (blockedHosts.includes(parsed.hostname)) {
                throw new Error('Access to localhost is not allowed');
            }
            
            // Block private IP ranges
            if (this.isPrivateIP(parsed.hostname)) {
                throw new Error('Access to private IP ranges is not allowed');
            }
            
            // Only allow HTTP/HTTPS
            if (!['http:', 'https:'].includes(parsed.protocol)) {
                throw new Error('Only HTTP and HTTPS protocols are allowed');
            }
            
            return true;
            
        } catch (error) {
            throw new Error(`Invalid URL: ${error.message}`);
        }
    }

    static isPrivateIP(hostname) {
        const privateRanges = [
            /^10\./,
            /^172\.(1[6-9]|2\d|3[01])\./,
            /^192\.168\./,
            /^169\.254\./,
            /^fc00:/,
            /^fd00:/
        ];
        
        return privateRanges.some(range => range.test(hostname));
    }

    /**
     * Safe HTTP client with whitelist
     */
    static async safeFetch(url, options = {}) {
        // Validate URL
        this.validateURL(url);
        
        // Whitelist of allowed domains
        const allowedDomains = [
            'api.emiratesnbd.com',
            'secure-api.partner.com'
        ];
        
        const parsed = new URL(url);
        if (!allowedDomains.some(domain => parsed.hostname.endsWith(domain))) {
            throw new Error('Domain not in whitelist');
        }
        
        // Set timeout
        const controller = new AbortController();
        const timeout = setTimeout(() => controller.abort(), 5000);
        
        try {
            const response = await fetch(url, {
                ...options,
                signal: controller.signal
            });
            
            return response;
        } finally {
            clearTimeout(timeout);
        }
    }
}

/**
 * API8:2023 - Security Misconfiguration
 * Secure HTTP headers and configurations
 */
class SecurityConfiguration {
    static setupSecurityHeaders(app) {
        // Use Helmet for security headers
        app.use(helmet({
            contentSecurityPolicy: {
                directives: {
                    defaultSrc: ["'self'"],
                    styleSrc: ["'self'", "'unsafe-inline'"],
                    scriptSrc: ["'self'"],
                    imgSrc: ["'self'", 'data:', 'https:'],
                    connectSrc: ["'self'"],
                    fontSrc: ["'self'"],
                    objectSrc: ["'none'"],
                    mediaSrc: ["'self'"],
                    frameSrc: ["'none'"]
                }
            },
            hsts: {
                maxAge: 31536000,
                includeSubDomains: true,
                preload: true
            },
            referrerPolicy: { policy: 'same-origin' },
            noSniff: true,
            xssFilter: true,
            hidePoweredBy: true
        }));
        
        // Custom security headers
        app.use((req, res, next) => {
            res.setHeader('X-API-Version', '1.0');
            res.setHeader('X-Response-Time', Date.now() - req.startTime);
            res.removeHeader('X-Powered-By');
            next();
        });
    }

    static configureCORS(app) {
        const cors = require('cors');
        
        const corsOptions = {
            origin: (origin, callback) => {
                const allowedOrigins = [
                    'https://emiratesnbd.com',
                    'https://app.emiratesnbd.com'
                ];
                
                if (!origin || allowedOrigins.includes(origin)) {
                    callback(null, true);
                } else {
                    callback(new Error('Not allowed by CORS'));
                }
            },
            credentials: true,
            optionsSuccessStatus: 200,
            methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
            allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID']
        };
        
        app.use(cors(corsOptions));
    }

    static disableUnnecessaryFeatures(app) {
        // Disable X-Powered-By header
        app.disable('x-powered-by');
        
        // Disable directory listing
        app.disable('view cache');
        
        // Disable ETag for sensitive endpoints
        app.set('etag', false);
    }
}

/**
 * API9:2023 - Improper Inventory Management
 * Track and monitor API endpoints
 */
class APIInventoryManager {
    constructor() {
        this.endpoints = new Map();
    }

    /**
     * Register API endpoint
     */
    registerEndpoint(method, path, metadata) {
        const key = `${method}:${path}`;
        this.endpoints.set(key, {
            method,
            path,
            ...metadata,
            registeredAt: new Date()
        });
    }

    /**
     * Get API inventory
     */
    getInventory() {
        return Array.from(this.endpoints.values());
    }

    /**
     * Detect deprecated endpoints
     */
    getDeprecatedEndpoints() {
        return this.getInventory().filter(endpoint => 
            endpoint.deprecated === true
        );
    }

    /**
     * Generate API documentation
     */
    generateDocumentation() {
        return {
            version: '1.0',
            endpoints: this.getInventory().map(endpoint => ({
                method: endpoint.method,
                path: endpoint.path,
                description: endpoint.description,
                authentication: endpoint.authentication,
                rateLimit: endpoint.rateLimit,
                deprecated: endpoint.deprecated
            }))
        };
    }
}

/**
 * API10:2023 - Unsafe Consumption of APIs
 * Validate responses from third-party APIs
 */
class ThirdPartyAPIProtection {
    /**
     * Validate response from external API
     */
    static async validateExternalResponse(response, expectedSchema) {
        const Ajv = require('ajv');
        const ajv = new Ajv();
        
        const validate = ajv.compile(expectedSchema);
        const valid = validate(response);
        
        if (!valid) {
            throw new Error(`Invalid API response: ${JSON.stringify(validate.errors)}`);
        }
        
        return true;
    }

    /**
     * Circuit breaker for external APIs
     */
    static createCircuitBreaker(serviceName, options = {}) {
        const CircuitBreaker = require('opossum');
        
        const breaker = new CircuitBreaker(this.callExternalAPI, {
            timeout: options.timeout || 3000,
            errorThresholdPercentage: options.errorThreshold || 50,
            resetTimeout: options.resetTimeout || 30000
        });
        
        breaker.on('open', () => {
            console.log(`Circuit breaker opened for ${serviceName}`);
        });
        
        breaker.on('halfOpen', () => {
            console.log(`Circuit breaker half-open for ${serviceName}`);
        });
        
        return breaker;
    }

    static async callExternalAPI(url, options) {
        const response = await fetch(url, options);
        if (!response.ok) {
            throw new Error(`API call failed: ${response.status}`);
        }
        return response.json();
    }
}

module.exports = {
    BOLAProtection,
    AuthenticationProtection,
    DataExposureProtection,
    ResourceProtection,
    FunctionAuthorizationProtection,
    BusinessFlowProtection,
    SSRFProtection,
    SecurityConfiguration,
    APIInventoryManager,
    ThirdPartyAPIProtection
};
```

---

## Q31: How do you implement end-to-end encryption for sensitive banking data in transit and at rest?

**Answer:**

Banking APIs must encrypt sensitive data both in transit (TLS/HTTPS) and at rest (database encryption) to protect customer information.

### Encryption Implementation:

```javascript
// encryption-manager.js
const crypto = require('crypto');
const fs = require('fs');

/**
 * Encryption Manager for Banking Data
 */
class EncryptionManager {
    constructor(config) {
        this.algorithm = 'aes-256-gcm';
        this.keyLength = 32; // 256 bits
        this.ivLength = 16;  // 128 bits
        this.saltLength = 64;
        this.tagLength = 16;
        
        // Load encryption keys from secure storage
        this.masterKey = this.loadMasterKey(config.masterKeyPath);
    }

    /**
     * Load master encryption key
     */
    loadMasterKey(keyPath) {
        if (!keyPath || !fs.existsSync(keyPath)) {
            throw new Error('Master encryption key not found');
        }
        
        const key = fs.readFileSync(keyPath);
        
        if (key.length !== this.keyLength) {
            throw new Error('Invalid master key length');
        }
        
        return key;
    }

    /**
     * Generate encryption key from password
     */
    deriveKey(password, salt) {
        return crypto.pbkdf2Sync(
            password,
            salt,
            100000, // iterations
            this.keyLength,
            'sha512'
        );
    }

    /**
     * Encrypt sensitive data
     */
    encrypt(plaintext) {
        try {
            // Generate random IV
            const iv = crypto.randomBytes(this.ivLength);
            
            // Create cipher
            const cipher = crypto.createCipheriv(this.algorithm, this.masterKey, iv);
            
            // Encrypt data
            let encrypted = cipher.update(plaintext, 'utf8', 'hex');
            encrypted += cipher.final('hex');
            
            // Get authentication tag
            const tag = cipher.getAuthTag();
            
            // Combine IV + tag + encrypted data
            const result = Buffer.concat([
                iv,
                tag,
                Buffer.from(encrypted, 'hex')
            ]).toString('base64');
            
            return result;
            
        } catch (error) {
            throw new Error(`Encryption failed: ${error.message}`);
        }
    }

    /**
     * Decrypt sensitive data
     */
    decrypt(ciphertext) {
        try {
            // Decode from base64
            const buffer = Buffer.from(ciphertext, 'base64');
            
            // Extract IV, tag, and encrypted data
            const iv = buffer.slice(0, this.ivLength);
            const tag = buffer.slice(this.ivLength, this.ivLength + this.tagLength);
            const encrypted = buffer.slice(this.ivLength + this.tagLength);
            
            // Create decipher
            const decipher = crypto.createDecipheriv(this.algorithm, this.masterKey, iv);
            decipher.setAuthTag(tag);
            
            // Decrypt data
            let decrypted = decipher.update(encrypted, undefined, 'utf8');
            decrypted += decipher.final('utf8');
            
            return decrypted;
            
        } catch (error) {
            throw new Error(`Decryption failed: ${error.message}`);
        }
    }

    /**
     * Encrypt object fields
     */
    encryptFields(obj, fieldsToEncrypt) {
        const encrypted = { ...obj };
        
        fieldsToEncrypt.forEach(field => {
            if (encrypted[field]) {
                encrypted[field] = this.encrypt(encrypted[field].toString());
            }
        });
        
        return encrypted;
    }

    /**
     * Decrypt object fields
     */
    decryptFields(obj, fieldsToDecrypt) {
        const decrypted = { ...obj };
        
        fieldsToDecrypt.forEach(field => {
            if (decrypted[field]) {
                decrypted[field] = this.decrypt(decrypted[field]);
            }
        });
        
        return decrypted;
    }

    /**
     * Hash password securely
     */
    async hashPassword(password) {
        const bcrypt = require('bcrypt');
        const saltRounds = 12;
        return bcrypt.hash(password, saltRounds);
    }

    /**
     * Verify password hash
     */
    async verifyPassword(password, hash) {
        const bcrypt = require('bcrypt');
        return bcrypt.compare(password, hash);
    }

    /**
     * Generate secure random token
     */
    generateToken(length = 32) {
        return crypto.randomBytes(length).toString('hex');
    }

    /**
     * Hash data with SHA-256
     */
    hash(data) {
        return crypto.createHash('sha256').update(data).digest('hex');
    }
}

/**
 * Database Encryption Layer
 */
class DatabaseEncryption {
    constructor(encryptionManager) {
        this.encryption = encryptionManager;
        this.sensitiveFields = {
            customers: ['ssn', 'passport_number', 'tax_id'],
            accounts: ['account_number', 'iban'],
            cards: ['card_number', 'cvv', 'pin']
        };
    }

    /**
     * Encrypt before saving to database
     */
    encryptBeforeSave(tableName, data) {
        const fieldsToEncrypt = this.sensitiveFields[tableName] || [];
        return this.encryption.encryptFields(data, fieldsToEncrypt);
    }

    /**
     * Decrypt after loading from database
     */
    decryptAfterLoad(tableName, data) {
        const fieldsToDecrypt = this.sensitiveFields[tableName] || [];
        return this.encryption.decryptFields(data, fieldsToDecrypt);
    }

    /**
     * PostgreSQL encryption functions
     */
    getPostgreSQLEncryptionConfig() {
        return `
            -- Enable pgcrypto extension
            CREATE EXTENSION IF NOT EXISTS pgcrypto;
            
            -- Encrypt column
            CREATE OR REPLACE FUNCTION encrypt_column(data TEXT)
            RETURNS TEXT AS $$
            BEGIN
                RETURN encode(pgp_sym_encrypt(data, '${process.env.DB_ENCRYPTION_KEY}'), 'base64');
            END;
            $$ LANGUAGE plpgsql;
            
            -- Decrypt column
            CREATE OR REPLACE FUNCTION decrypt_column(data TEXT)
            RETURNS TEXT AS $$
            BEGIN
                RETURN pgp_sym_decrypt(decode(data, 'base64'), '${process.env.DB_ENCRYPTION_KEY}');
            END;
            $$ LANGUAGE plpgsql;
            
            -- Example: Encrypt SSN before insert
            CREATE OR REPLACE FUNCTION encrypt_sensitive_data()
            RETURNS TRIGGER AS $$
            BEGIN
                IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
                    IF NEW.ssn IS NOT NULL THEN
                        NEW.ssn = encrypt_column(NEW.ssn);
                    END IF;
                    IF NEW.passport_number IS NOT NULL THEN
                        NEW.passport_number = encrypt_column(NEW.passport_number);
                    END IF;
                END IF;
                RETURN NEW;
            END;
            $$ LANGUAGE plpgsql;
            
            -- Create trigger
            CREATE TRIGGER encrypt_customer_data
            BEFORE INSERT OR UPDATE ON customers
            FOR EACH ROW
            EXECUTE FUNCTION encrypt_sensitive_data();
        `;
    }

    /**
     * Transparent Data Encryption (TDE) setup
     */
    setupTDE() {
        return `
            -- PostgreSQL TDE using pg_tde extension
            -- Requires PostgreSQL 15+ with pg_tde extension
            
            -- 1. Initialize TDE
            SELECT pg_tde_add_key_provider_file('file-vault', '/etc/postgresql/encryption/tde-vault.per');
            SELECT pg_tde_set_master_key('my-master-key', 'file-vault');
            
            -- 2. Create encrypted database
            CREATE DATABASE enbd_banking_encrypted
            WITH TEMPLATE = template0
            ENCRYPTION_METHOD = 'aes-256-ctr';
            
            -- 3. All data in this database is automatically encrypted at rest
        `;
    }
}

/**
 * TLS/HTTPS Configuration
 */
class TLSConfiguration {
    /**
     * Generate self-signed certificate (development only)
     */
    static generateSelfSignedCert() {
        const { execSync } = require('child_process');
        
        const commands = `
            # Generate private key
            openssl genrsa -out server.key 2048
            
            # Generate CSR
            openssl req -new -key server.key -out server.csr \
                -subj "/C=AE/ST=Dubai/L=Dubai/O=Emirates NBD/CN=api.emiratesnbd.com"
            
            # Generate self-signed certificate
            openssl x509 -req -days 365 -in server.csr \
                -signkey server.key -out server.crt
        `;
        
        console.log('Generate self-signed certificate (dev only):', commands);
    }

    /**
     * HTTPS server configuration
     */
    static createHTTPSServer(app) {
        const https = require('https');
        const fs = require('fs');
        
        const options = {
            key: fs.readFileSync('./certs/server.key'),
            cert: fs.readFileSync('./certs/server.crt'),
            
            // Strong cipher suites
            ciphers: [
                'ECDHE-ECDSA-AES256-GCM-SHA384',
                'ECDHE-RSA-AES256-GCM-SHA384',
                'ECDHE-ECDSA-CHACHA20-POLY1305',
                'ECDHE-RSA-CHACHA20-POLY1305',
                'ECDHE-ECDSA-AES128-GCM-SHA256',
                'ECDHE-RSA-AES128-GCM-SHA256'
            ].join(':'),
            
            // TLS version
            minVersion: 'TLSv1.2',
            maxVersion: 'TLSv1.3',
            
            // Prefer server cipher order
            honorCipherOrder: true,
            
            // HSTS
            requestCert: false,
            rejectUnauthorized: true
        };
        
        const server = https.createServer(options, app);
        
        return server;
    }

    /**
     * Mutual TLS (mTLS) configuration
     */
    static createMTLSServer(app) {
        const https = require('https');
        const fs = require('fs');
        
        const options = {
            key: fs.readFileSync('./certs/server.key'),
            cert: fs.readFileSync('./certs/server.crt'),
            ca: fs.readFileSync('./certs/ca.crt'),
            
            // Require client certificate
            requestCert: true,
            rejectUnauthorized: true
        };
        
        const server = https.createServer(options, app);
        
        // Verify client certificate
        server.on('secureConnection', (tlsSocket) => {
            const cert = tlsSocket.getPeerCertificate();
            
            if (!cert || !cert.subject) {
                tlsSocket.destroy();
                return;
            }
            
            console.log('Client certificate:', cert.subject.CN);
        });
        
        return server;
    }
}

/**
 * Field-Level Encryption Service
 */
class FieldLevelEncryption {
    constructor(encryptionManager) {
        this.encryption = encryptionManager;
    }

    /**
     * Encrypt credit card number
     */
    encryptCardNumber(cardNumber) {
        // Remove spaces and dashes
        const cleaned = cardNumber.replace(/[\s-]/g, '');
        
        // Validate card number (Luhn algorithm)
        if (!this.validateCardNumber(cleaned)) {
            throw new Error('Invalid card number');
        }
        
        // Encrypt
        const encrypted = this.encryption.encrypt(cleaned);
        
        // Store last 4 digits in plain text for display
        const last4 = cleaned.slice(-4);
        
        return {
            encrypted,
            last4,
            masked: `****-****-****-${last4}`
        };
    }

    /**
     * Validate card number using Luhn algorithm
     */
    validateCardNumber(cardNumber) {
        let sum = 0;
        let isEven = false;
        
        for (let i = cardNumber.length - 1; i >= 0; i--) {
            let digit = parseInt(cardNumber[i]);
            
            if (isEven) {
                digit *= 2;
                if (digit > 9) digit -= 9;
            }
            
            sum += digit;
            isEven = !isEven;
        }
        
        return sum % 10 === 0;
    }

    /**
     * Encrypt sensitive account data
     */
    async encryptAccountData(account) {
        return {
            id: account.id,
            customerId: account.customerId,
            accountNumber: this.encryption.encrypt(account.accountNumber),
            iban: this.encryption.encrypt(account.iban),
            balance: account.balance, // Not encrypted (needed for queries)
            encryptedAt: new Date().toISOString()
        };
    }

    /**
     * Tokenize sensitive data
     */
    tokenize(sensitiveData) {
        const token = this.encryption.generateToken();
        
        // Store mapping in secure vault (Redis/Vault)
        redis.setex(`token:${token}`, 3600, this.encryption.encrypt(sensitiveData));
        
        return token;
    }

    /**
     * Detokenize
     */
    async detokenize(token) {
        const encrypted = await redis.get(`token:${token}`);
        
        if (!encrypted) {
            throw new Error('Invalid or expired token');
        }
        
        return this.encryption.decrypt(encrypted);
    }
}

// Usage Example
const encryptionManager = new EncryptionManager({
    masterKeyPath: './keys/master.key'
});

const dbEncryption = new DatabaseEncryption(encryptionManager);
const fieldEncryption = new FieldLevelEncryption(encryptionManager);

// Encrypt customer data before saving
const customerData = {
    name: 'John Doe',
    email: 'john@example.com',
    ssn: '123-45-6789',
    passport_number: 'AB1234567'
};

const encrypted = dbEncryption.encryptBeforeSave('customers', customerData);
// Save to database...

// Decrypt after loading
const decrypted = dbEncryption.decryptAfterLoad('customers', encrypted);

module.exports = {
    EncryptionManager,
    DatabaseEncryption,
    TLSConfiguration,
    FieldLevelEncryption
};
```

---

## Q32: How do you implement secure API key management and rotation for banking systems?

**Answer:**

Proper API key management prevents unauthorized access and ensures keys can be rotated without service disruption.

### API Key Management:

```javascript
// api-key-manager.js
const crypto = require('crypto');
const { Pool } = require('pg');

/**
 * API Key Manager
 */
class APIKeyManager {
    constructor(db) {
        this.db = db;
        this.keyPrefix = 'enbd_';
        this.keyLength = 32;
    }

    /**
     * Generate API key
     */
    generateAPIKey() {
        const randomBytes = crypto.randomBytes(this.keyLength);
        const key = `${this.keyPrefix}${randomBytes.toString('base64url')}`;
        return key;
    }

    /**
     * Hash API key for storage
     */
    hashAPIKey(apiKey) {
        return crypto
            .createHash('sha256')
            .update(apiKey)
            .digest('hex');
    }

    /**
     * Create API key for client
     */
    async createAPIKey(clientData) {
        const {
            clientId,
            clientName,
            permissions = [],
            rateLimit = 1000,
            expiresIn = 365 // days
        } = clientData;
        
        // Generate key
        const apiKey = this.generateAPIKey();
        const hashedKey = this.hashAPIKey(apiKey);
        
        // Calculate expiry
        const expiresAt = new Date();
        expiresAt.setDate(expiresAt.getDate() + expiresIn);
        
        // Store in database
        const result = await this.db.query(`
            INSERT INTO api_keys (
                client_id,
                client_name,
                key_hash,
                permissions,
                rate_limit,
                expires_at,
                created_at
            ) VALUES ($1, $2, $3, $4, $5, $6, NOW())
            RETURNING id
        `, [
            clientId,
            clientName,
            hashedKey,
            JSON.stringify(permissions),
            rateLimit,
            expiresAt
        ]);
        
        const keyId = result.rows[0].id;
        
        // Log key creation
        await this.logKeyEvent(keyId, 'created', { clientId, clientName });
        
        // Return plaintext key (only time it's visible)
        return {
            keyId,
            apiKey, // Show this ONCE to the client
            expiresAt,
            permissions,
            rateLimit
        };
    }

    /**
     * Verify API key
     */
    async verifyAPIKey(apiKey) {
        const hashedKey = this.hashAPIKey(apiKey);
        
        const result = await this.db.query(`
            SELECT 
                id,
                client_id,
                client_name,
                permissions,
                rate_limit,
                expires_at,
                is_active,
                last_used_at
            FROM api_keys
            WHERE key_hash = $1
            AND is_active = true
            AND (expires_at IS NULL OR expires_at > NOW())
        `, [hashedKey]);
        
        if (result.rows.length === 0) {
            await this.logKeyEvent(null, 'invalid_attempt', { hashedKey });
            return null;
        }
        
        const keyData = result.rows[0];
        
        // Update last used timestamp
        await this.db.query(`
            UPDATE api_keys
            SET last_used_at = NOW(),
                usage_count = usage_count + 1
            WHERE id = $1
        `, [keyData.id]);
        
        return {
            keyId: keyData.id,
            clientId: keyData.client_id,
            clientName: keyData.client_name,
            permissions: JSON.parse(keyData.permissions),
            rateLimit: keyData.rate_limit
        };
    }

    /**
     * Rotate API key
     */
    async rotateAPIKey(oldApiKey) {
        // Verify old key
        const keyData = await this.verifyAPIKey(oldApiKey);
        
        if (!keyData) {
            throw new Error('Invalid API key');
        }
        
        // Get client info
        const client = await this.db.query(
            'SELECT * FROM api_keys WHERE id = $1',
            [keyData.keyId]
        );
        
        if (client.rows.length === 0) {
            throw new Error('Client not found');
        }
        
        const clientData = client.rows[0];
        
        // Generate new key
        const newKey = await this.createAPIKey({
            clientId: clientData.client_id,
            clientName: clientData.client_name,
            permissions: JSON.parse(clientData.permissions),
            rateLimit: clientData.rate_limit
        });
        
        // Mark old key as rotated (keep for grace period)
        await this.db.query(`
            UPDATE api_keys
            SET is_active = false,
                rotated_at = NOW(),
                rotated_to_id = $1
            WHERE id = $2
        `, [newKey.keyId, keyData.keyId]);
        
        await this.logKeyEvent(keyData.keyId, 'rotated', { 
            newKeyId: newKey.keyId 
        });
        
        return newKey;
    }

    /**
     * Revoke API key
     */
    async revokeAPIKey(apiKey, reason) {
        const hashedKey = this.hashAPIKey(apiKey);
        
        const result = await this.db.query(`
            UPDATE api_keys
            SET is_active = false,
                revoked_at = NOW(),
                revoked_reason = $2
            WHERE key_hash = $1
            RETURNING id, client_id
        `, [hashedKey, reason]);
        
        if (result.rows.length === 0) {
            throw new Error('API key not found');
        }
        
        const keyData = result.rows[0];
        
        await this.logKeyEvent(keyData.id, 'revoked', { reason });
        
        return { revoked: true, keyId: keyData.id };
    }

    /**
     * List API keys for client
     */
    async listClientKeys(clientId) {
        const result = await this.db.query(`
            SELECT 
                id,
                client_name,
                LEFT(key_hash, 8) as key_preview,
                permissions,
                rate_limit,
                is_active,
                created_at,
                expires_at,
                last_used_at,
                usage_count
            FROM api_keys
            WHERE client_id = $1
            ORDER BY created_at DESC
        `, [clientId]);
        
        return result.rows.map(row => ({
            keyId: row.id,
            keyPreview: `${this.keyPrefix}${row.key_preview}...`,
            permissions: JSON.parse(row.permissions),
            rateLimit: row.rate_limit,
            isActive: row.is_active,
            createdAt: row.created_at,
            expiresAt: row.expires_at,
            lastUsedAt: row.last_used_at,
            usageCount: row.usage_count
        }));
    }

    /**
     * Check key expiry and send notifications
     */
    async checkExpiringKeys() {
        const result = await this.db.query(`
            SELECT 
                id,
                client_id,
                client_name,
                expires_at
            FROM api_keys
            WHERE is_active = true
            AND expires_at > NOW()
            AND expires_at < NOW() + INTERVAL '30 days'
        `);
        
        for (const key of result.rows) {
            const daysUntilExpiry = Math.floor(
                (new Date(key.expires_at) - new Date()) / (1000 * 60 * 60 * 24)
            );
            
            // Send notification
            console.log(`⚠️ API key ${key.id} expires in ${daysUntilExpiry} days`);
            
            // Send email/notification to client
            await this.sendExpiryNotification(key.client_id, key.id, daysUntilExpiry);
        }
    }

    /**
     * Auto-rotate keys before expiry
     */
    async autoRotateExpiringKeys() {
        const result = await this.db.query(`
            SELECT 
                id,
                key_hash,
                client_id
            FROM api_keys
            WHERE is_active = true
            AND expires_at > NOW()
            AND expires_at < NOW() + INTERVAL '7 days'
            AND auto_rotate = true
        `);
        
        for (const key of result.rows) {
            try {
                // Rotate key automatically
                await this.rotateAPIKey(key.key_hash);
                
                console.log(`✅ Auto-rotated key ${key.id}`);
                
                // Notify client of new key
                await this.sendRotationNotification(key.client_id, key.id);
                
            } catch (error) {
                console.error(`Failed to auto-rotate key ${key.id}:`, error);
            }
        }
    }

    /**
     * Log key events for audit
     */
    async logKeyEvent(keyId, eventType, metadata = {}) {
        await this.db.query(`
            INSERT INTO api_key_audit_log (
                key_id,
                event_type,
                metadata,
                created_at
            ) VALUES ($1, $2, $3, NOW())
        `, [keyId, eventType, JSON.stringify(metadata)]);
    }

    async sendExpiryNotification(clientId, keyId, daysUntilExpiry) {
        // Implement notification logic
        console.log(`Send expiry notification to client ${clientId}`);
    }

    async sendRotationNotification(clientId, keyId) {
        // Implement notification logic
        console.log(`Send rotation notification to client ${clientId}`);
    }
}

/**
 * API Key Middleware
 */
class APIKeyMiddleware {
    constructor(apiKeyManager) {
        this.apiKeyManager = apiKeyManager;
    }

    /**
     * Verify API key from request
     */
    authenticate() {
        return async (req, res, next) => {
            try {
                // Get API key from header
                const apiKey = req.header('X-API-Key') || req.header('Authorization')?.replace('Bearer ', '');
                
                if (!apiKey) {
                    return res.status(401).json({
                        error: 'Unauthorized',
                        message: 'API key required'
                    });
                }
                
                // Verify key
                const keyData = await this.apiKeyManager.verifyAPIKey(apiKey);
                
                if (!keyData) {
                    return res.status(401).json({
                        error: 'Unauthorized',
                        message: 'Invalid or expired API key'
                    });
                }
                
                // Attach key data to request
                req.apiKey = keyData;
                req.client = {
                    id: keyData.clientId,
                    name: keyData.clientName
                };
                
                next();
                
            } catch (error) {
                res.status(500).json({
                    error: 'InternalServerError',
                    message: 'Authentication failed'
                });
            }
        };
    }

    /**
     * Check API key permissions
     */
    requirePermission(...requiredPermissions) {
        return (req, res, next) => {
            if (!req.apiKey) {
                return res.status(401).json({
                    error: 'Unauthorized',
                    message: 'Authentication required'
                });
            }
            
            const hasPermissions = requiredPermissions.every(perm =>
                req.apiKey.permissions.includes(perm)
            );
            
            if (!hasPermissions) {
                return res.status(403).json({
                    error: 'Forbidden',
                    message: `Missing permissions: ${requiredPermissions.join(', ')}`
                });
            }
            
            next();
        };
    }
}

/**
 * Database Schema
 */
const schema = `
CREATE TABLE api_keys (
    id SERIAL PRIMARY KEY,
    client_id VARCHAR(100) NOT NULL,
    client_name VARCHAR(255) NOT NULL,
    key_hash VARCHAR(64) UNIQUE NOT NULL,
    permissions JSONB DEFAULT '[]',
    rate_limit INTEGER DEFAULT 1000,
    is_active BOOLEAN DEFAULT true,
    auto_rotate BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    last_used_at TIMESTAMP,
    usage_count INTEGER DEFAULT 0,
    revoked_at TIMESTAMP,
    revoked_reason TEXT,
    rotated_at TIMESTAMP,
    rotated_to_id INTEGER REFERENCES api_keys(id)
);

CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_client ON api_keys(client_id);
CREATE INDEX idx_api_keys_expires ON api_keys(expires_at) WHERE is_active = true;

CREATE TABLE api_key_audit_log (
    id SERIAL PRIMARY KEY,
    key_id INTEGER REFERENCES api_keys(id),
    event_type VARCHAR(50) NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_log_key ON api_key_audit_log(key_id);
CREATE INDEX idx_audit_log_created ON api_key_audit_log(created_at);
`;

// Usage Example
const pool = new Pool({
    host: 'localhost',
    database: 'enbd_banking',
    user: 'postgres',
    password: process.env.DB_PASSWORD
});

const apiKeyManager = new APIKeyManager(pool);
const apiKeyMiddleware = new APIKeyMiddleware(apiKeyManager);

// Express routes
const express = require('express');
const app = express();

// Protected route
app.get('/api/accounts',
    apiKeyMiddleware.authenticate(),
    apiKeyMiddleware.requirePermission('accounts:read'),
    async (req, res) => {
        // Access accounts
        res.json({ accounts: [] });
    }
);

// Schedule key expiry checks (daily)
setInterval(() => {
    apiKeyManager.checkExpiringKeys();
    apiKeyManager.autoRotateExpiringKeys();
}, 24 * 60 * 60 * 1000);

module.exports = {
    APIKeyManager,
    APIKeyMiddleware
};
```

---
