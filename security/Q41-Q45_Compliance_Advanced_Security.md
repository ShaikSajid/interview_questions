# Security Questions 41-45: Compliance & Advanced Security

---

### Q41. How do you implement GDPR compliance in Node.js applications?

**Answer:**

```javascript
/**
 * GDPR Compliance Service
 */
class GDPRComplianceService {
  constructor() {
    this.consentStore = new Map();
  }
  
  /**
   * Record user consent (Art. 7 GDPR)
   */
  async recordConsent(userId, consentData) {
    const consent = {
      userId,
      consentId: crypto.randomBytes(16).toString('hex'),
      purpose: consentData.purpose, // 'marketing', 'analytics', 'necessary'
      granted: consentData.granted,
      timestamp: new Date(),
      ipAddress: consentData.ip,
      userAgent: consentData.userAgent,
      version: consentData.version // Consent text version
    };
    
    await this.storeConsent(consent);
    
    return consent.consentId;
  }
  
  /**
   * Check if user has given consent
   */
  async hasConsent(userId, purpose) {
    const consents = await this.getUserConsents(userId);
    
    const relevantConsent = consents.find(c => 
      c.purpose === purpose && c.granted
    );
    
    return !!relevantConsent;
  }
  
  /**
   * Right to Access (Art. 15 GDPR)
   */
  async exportUserData(userId) {
    const userData = {
      personalInfo: await this.getPersonalInfo(userId),
      accounts: await this.getAccounts(userId),
      transactions: await this.getTransactions(userId),
      consents: await this.getUserConsents(userId),
      loginHistory: await this.getLoginHistory(userId),
      activityLog: await this.getActivityLog(userId)
    };
    
    return {
      userId,
      exportedAt: new Date(),
      data: userData,
      format: 'JSON'
    };
  }
  
  /**
   * Right to Erasure (Art. 17 GDPR - "Right to be Forgotten")
   */
  async deleteUserData(userId, retentionOverride = false) {
    // Check retention requirements
    if (!retentionOverride) {
      const retention = await this.checkRetentionRequirements(userId);
      if (retention.mustRetain) {
        throw new Error(`Cannot delete: ${retention.reason}`);
      }
    }
    
    // Anonymize instead of hard delete (for audit trail)
    await this.anonymizeUser(userId);
    
    // Delete non-essential data
    await this.deleteNonEssentialData(userId);
    
    // Log deletion request (required for compliance)
    await this.logDataDeletion(userId);
    
    return {
      success: true,
      message: 'User data deleted/anonymized'
    };
  }
  
  /**
   * Right to Rectification (Art. 16 GDPR)
   */
  async updateUserData(userId, updates, requestedBy) {
    // Verify identity
    if (requestedBy !== userId) {
      throw new Error('Unauthorized data modification');
    }
    
    // Validate updates
    const validated = await this.validateUpdates(updates);
    
    // Apply updates
    await this.applyUpdates(userId, validated);
    
    // Log changes
    await this.logDataModification(userId, updates);
    
    return {
      success: true,
      message: 'Personal data updated'
    };
  }
  
  /**
   * Right to Data Portability (Art. 20 GDPR)
   */
  async exportPortableData(userId, format = 'JSON') {
    const data = await this.exportUserData(userId);
    
    // Convert to requested format
    switch (format.toUpperCase()) {
      case 'JSON':
        return JSON.stringify(data, null, 2);
      
      case 'CSV':
        return this.convertToCSV(data);
      
      case 'XML':
        return this.convertToXML(data);
      
      default:
        throw new Error('Unsupported format');
    }
  }
  
  /**
   * Right to Object (Art. 21 GDPR)
   */
  async recordObjection(userId, objectionData) {
    await this.storeObjection({
      userId,
      purpose: objectionData.purpose, // 'direct-marketing', 'profiling'
      timestamp: new Date(),
      reason: objectionData.reason
    });
    
    // Stop processing immediately
    await this.stopProcessing(userId, objectionData.purpose);
    
    return {
      success: true,
      message: 'Objection recorded'
    };
  }
  
  /**
   * Data Minimization (Art. 5 GDPR)
   */
  validateDataCollection(data) {
    const essentialFields = [
      'email',
      'name',
      'dateOfBirth'
    ];
    
    const collectedFields = Object.keys(data);
    const excessFields = collectedFields.filter(f => !essentialFields.includes(f));
    
    if (excessFields.length > 0) {
      console.warn('Collecting non-essential data:', excessFields);
    }
    
    return {
      valid: true,
      warnings: excessFields
    };
  }
  
  /**
   * Purpose Limitation (Art. 5 GDPR)
   */
  async checkPurposeLimitation(userId, data, purpose) {
    const consent = await this.hasConsent(userId, purpose);
    
    if (!consent) {
      throw new Error(`No consent for purpose: ${purpose}`);
    }
    
    return true;
  }
  
  /**
   * Data Retention Policy
   */
  async checkRetentionRequirements(userId) {
    const user = await this.getUser(userId);
    
    // Financial records: 7 years (regulatory requirement)
    const hasActiveAccounts = await this.hasActiveAccounts(userId);
    if (hasActiveAccounts) {
      return {
        mustRetain: true,
        reason: 'Active accounts exist'
      };
    }
    
    const lastTransaction = await this.getLastTransaction(userId);
    if (lastTransaction) {
      const daysSince = (Date.now() - lastTransaction.date) / (1000 * 60 * 60 * 24);
      if (daysSince < 7 * 365) { // 7 years
        return {
          mustRetain: true,
          reason: 'Financial records must be retained for 7 years'
        };
      }
    }
    
    return {
      mustRetain: false
    };
  }
  
  /**
   * Anonymization (not deletion)
   */
  async anonymizeUser(userId) {
    await this.updateUser(userId, {
      email: `deleted-${userId}@anonymized.local`,
      name: '[DELETED]',
      phoneNumber: null,
      address: null,
      dateOfBirth: null,
      // Keep only: userId (for referential integrity)
    });
  }
  
  /**
   * Data Breach Notification (Art. 33 & 34 GDPR)
   */
  async notifyDataBreach(breachDetails) {
    const affectedUsers = await this.getAffectedUsers(breachDetails);
    
    // Notify supervisory authority within 72 hours
    if (breachDetails.severity === 'high') {
      await this.notifySupervisoryAuthority({
        nature: breachDetails.nature,
        affectedUsersCount: affectedUsers.length,
        likelyConsequences: breachDetails.consequences,
        measures: breachDetails.measures,
        dpoContact: process.env.DPO_CONTACT
      });
    }
    
    // Notify affected users without undue delay
    if (breachDetails.highRiskToRights) {
      for (const user of affectedUsers) {
        await this.notifyUser(user, {
          nature: breachDetails.nature,
          consequences: breachDetails.consequences,
          measures: breachDetails.measures
        });
      }
    }
    
    // Document breach
    await this.documentBreach(breachDetails);
  }
  
  async storeConsent(consent) {
    // Store in database
  }
  
  async getUserConsents(userId) {
    return [];
  }
  
  async getPersonalInfo(userId) {
    return {};
  }
  
  async getAccounts(userId) {
    return [];
  }
  
  async getTransactions(userId) {
    return [];
  }
  
  async getLoginHistory(userId) {
    return [];
  }
  
  async getActivityLog(userId) {
    return [];
  }
  
  async deleteNonEssentialData(userId) {
    // Delete logic
  }
  
  async logDataDeletion(userId) {
    // Audit log
  }
  
  async validateUpdates(updates) {
    return updates;
  }
  
  async applyUpdates(userId, updates) {
    // Update logic
  }
  
  async logDataModification(userId, updates) {
    // Audit log
  }
  
  convertToCSV(data) {
    return '';
  }
  
  convertToXML(data) {
    return '';
  }
  
  async storeObjection(objection) {
    // Store objection
  }
  
  async stopProcessing(userId, purpose) {
    // Stop processing
  }
  
  async getUser(userId) {
    return {};
  }
  
  async hasActiveAccounts(userId) {
    return false;
  }
  
  async getLastTransaction(userId) {
    return null;
  }
  
  async updateUser(userId, data) {
    // Update user
  }
  
  async getAffectedUsers(breach) {
    return [];
  }
  
  async notifySupervisoryAuthority(details) {
    // Notify authority
  }
  
  async notifyUser(user, details) {
    // Notify user
  }
  
  async documentBreach(details) {
    // Document breach
  }
}

/**
 * Cookie Consent Management
 */
class CookieConsentService {
  /**
   * Set cookie consent preferences
   */
  setCookieConsent(res, preferences) {
    res.cookie('cookie_consent', JSON.stringify({
      necessary: true, // Always true
      functional: preferences.functional,
      analytics: preferences.analytics,
      marketing: preferences.marketing,
      timestamp: Date.now()
    }), {
      maxAge: 365 * 24 * 60 * 60 * 1000, // 1 year
      httpOnly: true,
      secure: true,
      sameSite: 'strict'
    });
  }
  
  /**
   * Check cookie consent
   */
  hasAnalyticsConsent(req) {
    const consent = req.cookies.cookie_consent;
    if (!consent) return false;
    
    try {
      const preferences = JSON.parse(consent);
      return preferences.analytics === true;
    } catch (error) {
      return false;
    }
  }
}

module.exports = {
  GDPRComplianceService,
  CookieConsentService
};
```

**GDPR Compliance Checklist:**

1. ✅ **Lawful basis for processing** (consent, contract, legal obligation)
2. ✅ **Right to access** (Art. 15) - data export
3. ✅ **Right to erasure** (Art. 17) - delete/anonymize
4. ✅ **Right to rectification** (Art. 16) - update data
5. ✅ **Right to portability** (Art. 20) - export in standard format
6. ✅ **Right to object** (Art. 21) - stop processing
7. ✅ **Data minimization** - collect only necessary data
8. ✅ **Purpose limitation** - use data only for stated purpose
9. ✅ **Storage limitation** - retention policies
10. ✅ **Breach notification** - within 72 hours

---

### Q42. How do you implement secure logging without exposing sensitive data?

**Answer:**

```javascript
const winston = require('winston');

/**
 * Secure Logging Service
 */
class SecureLoggingService {
  constructor() {
    this.sensitiveFields = [
      'password',
      'passwordHash',
      'ssn',
      'creditCard',
      'cardNumber',
      'cvv',
      'pin',
      'token',
      'secret',
      'apiKey',
      'privateKey'
    ];
    
    this.setupLogger();
  }
  
  /**
   * Setup Winston logger with security
   */
  setupLogger() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        this.sanitizeFormat(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error',
          maxsize: 10485760, // 10MB
          maxFiles: 10
        }),
        new winston.transports.File({
          filename: 'logs/combined.log',
          maxsize: 10485760,
          maxFiles: 10
        })
      ]
    });
    
    if (process.env.NODE_ENV !== 'production') {
      this.logger.add(new winston.transports.Console({
        format: winston.format.simple()
      }));
    }
  }
  
  /**
   * Sanitize format to redact sensitive data
   */
  sanitizeFormat() {
    return winston.format((info) => {
      return this.sanitizeObject(info);
    })();
  }
  
  /**
   * Sanitize object recursively
   */
  sanitizeObject(obj) {
    if (typeof obj !== 'object' || obj === null) {
      return obj;
    }
    
    if (Array.isArray(obj)) {
      return obj.map(item => this.sanitizeObject(item));
    }
    
    const sanitized = {};
    
    for (const [key, value] of Object.entries(obj)) {
      const lowerKey = key.toLowerCase();
      
      // Redact sensitive fields
      if (this.isSensitiveField(lowerKey)) {
        sanitized[key] = '[REDACTED]';
      }
      // Mask card numbers
      else if (lowerKey.includes('card') && typeof value === 'string') {
        sanitized[key] = this.maskCardNumber(value);
      }
      // Mask email
      else if (lowerKey.includes('email') && typeof value === 'string') {
        sanitized[key] = this.maskEmail(value);
      }
      // Recursive sanitization
      else if (typeof value === 'object') {
        sanitized[key] = this.sanitizeObject(value);
      }
      else {
        sanitized[key] = value;
      }
    }
    
    return sanitized;
  }
  
  /**
   * Check if field is sensitive
   */
  isSensitiveField(fieldName) {
    return this.sensitiveFields.some(sensitive => 
      fieldName.includes(sensitive.toLowerCase())
    );
  }
  
  /**
   * Mask card number
   */
  maskCardNumber(cardNumber) {
    if (typeof cardNumber !== 'string' || cardNumber.length < 4) {
      return '[REDACTED]';
    }
    return `****-****-****-${cardNumber.slice(-4)}`;
  }
  
  /**
   * Mask email
   */
  maskEmail(email) {
    if (typeof email !== 'string' || !email.includes('@')) {
      return '[REDACTED]';
    }
    const [local, domain] = email.split('@');
    return `${local.substring(0, 2)}***@${domain}`;
  }
  
  /**
   * Mask phone number
   */
  maskPhoneNumber(phone) {
    if (typeof phone !== 'string' || phone.length < 4) {
      return '[REDACTED]';
    }
    return `***-***-${phone.slice(-4)}`;
  }
  
  /**
   * Safe logging methods
   */
  info(message, metadata = {}) {
    this.logger.info(message, this.sanitizeObject(metadata));
  }
  
  error(message, error, metadata = {}) {
    // Don't log stack traces in production with sensitive data
    const sanitizedError = {
      message: error.message,
      name: error.name,
      ...(process.env.NODE_ENV !== 'production' && { stack: error.stack })
    };
    
    this.logger.error(message, {
      error: sanitizedError,
      ...this.sanitizeObject(metadata)
    });
  }
  
  warn(message, metadata = {}) {
    this.logger.warn(message, this.sanitizeObject(metadata));
  }
  
  debug(message, metadata = {}) {
    if (process.env.NODE_ENV !== 'production') {
      this.logger.debug(message, this.sanitizeObject(metadata));
    }
  }
  
  /**
   * Audit log (never redact - separate secure storage)
   */
  audit(event, userId, details) {
    // Store in separate audit log with encryption
    const auditEntry = {
      event,
      userId,
      timestamp: new Date(),
      ip: details.ip,
      userAgent: details.userAgent,
      details: details.data // Not sanitized
    };
    
    // Store encrypted
    this.storeAuditLog(auditEntry);
  }
  
  async storeAuditLog(entry) {
    // Store in secure audit database
  }
}

/**
 * Request Logging Middleware (Secure)
 */
class SecureRequestLoggingMiddleware {
  constructor() {
    this.logger = new SecureLoggingService();
  }
  
  middleware() {
    return (req, res, next) => {
      const startTime = Date.now();
      
      // Capture response
      const originalSend = res.send;
      res.send = function(data) {
        res.send = originalSend;
        
        const duration = Date.now() - startTime;
        
        setImmediate(() => {
          this.logger.info('HTTP Request', {
            method: req.method,
            path: req.path,
            statusCode: res.statusCode,
            duration,
            userId: req.user?.id,
            ip: req.ip,
            // Don't log request body (may contain sensitive data)
            // Don't log response body
          });
        });
        
        return res.send(data);
      }.bind(this);
      
      next();
    };
  }
}

module.exports = {
  SecureLoggingService,
  SecureRequestLoggingMiddleware
};
```

**Secure Logging Best Practices:**

1. ✅ **Never log passwords or secrets**
2. ✅ **Mask PII** (emails, phone numbers, SSN)
3. ✅ **Mask card numbers** (show only last 4 digits)
4. ✅ **Separate audit logs** (encrypted storage)
5. ✅ **Log rotation** and size limits
6. ✅ **Centralized logging** with encryption in transit
7. ✅ **Access control** for log files

---

### Q43. How do you implement security headers and browser security?

**Answer:**

```javascript
const helmet = require('helmet');

/**
 * Browser Security Headers Service
 */
class BrowserSecurityService {
  /**
   * Configure all security headers
   */
  static configureSecurityHeaders(app) {
    // Use Helmet with strict settings
    app.use(helmet({
      // Content Security Policy
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          scriptSrc: ["'self'", "'nonce-{NONCE}'"],
          styleSrc: ["'self'", "'nonce-{NONCE}'"],
          imgSrc: ["'self'", "data:", "https:"],
          fontSrc: ["'self'"],
          connectSrc: ["'self'", "https://api.enbd.com"],
          frameSrc: ["'none'"],
          objectSrc: ["'none'"],
          baseUri: ["'self'"],
          formAction: ["'self'"],
          frameAncestors: ["'none'"],
          upgradeInsecureRequests: []
        }
      },
      
      // Strict Transport Security (HSTS)
      hsts: {
        maxAge: 31536000, // 1 year
        includeSubDomains: true,
        preload: true
      },
      
      // X-Frame-Options
      frameguard: {
        action: 'deny'
      },
      
      // X-Content-Type-Options
      noSniff: true,
      
      // X-XSS-Protection
      xssFilter: true,
      
      // Referrer-Policy
      referrerPolicy: {
        policy: 'strict-origin-when-cross-origin'
      },
      
      // Remove X-Powered-By
      hidePoweredBy: true,
      
      // DNS Prefetch Control
      dnsPrefetchControl: {
        allow: false
      },
      
      // IE No Open
      ieNoOpen: true,
      
      // Don't sniff MIME types
      noCache: false
    }));
    
    // Additional custom headers
    app.use((req, res, next) => {
      // Permissions Policy (formerly Feature Policy)
      res.setHeader('Permissions-Policy', 
        'geolocation=(), microphone=(), camera=(), payment=(self)'
      );
      
      // Cross-Origin Policies
      res.setHeader('Cross-Origin-Embedder-Policy', 'require-corp');
      res.setHeader('Cross-Origin-Opener-Policy', 'same-origin');
      res.setHeader('Cross-Origin-Resource-Policy', 'same-origin');
      
      // Clear-Site-Data (for logout)
      if (req.path === '/api/auth/logout') {
        res.setHeader('Clear-Site-Data', '"cache", "cookies", "storage"');
      }
      
      next();
    });
  }
  
  /**
   * Subresource Integrity (SRI)
   */
  static getSRIHash(content) {
    const hash = crypto.createHash('sha384').update(content).digest('base64');
    return `sha384-${hash}`;
  }
  
  /**
   * Trusted Types Policy
   */
  static getTrustedTypesPolicy() {
    return `
      <meta http-equiv="Content-Security-Policy" 
            content="require-trusted-types-for 'script'; trusted-types default">
      <script>
        if (window.trustedTypes && trustedTypes.createPolicy) {
          trustedTypes.createPolicy('default', {
            createHTML: (string) => DOMPurify.sanitize(string),
            createScriptURL: (string) => string,
            createScript: (string) => string
          });
        }
      </script>
    `;
  }
}

module.exports = {
  BrowserSecurityService
};
```

**Browser Security Headers:**

1. ✅ **HSTS** - Force HTTPS
2. ✅ **CSP** - Content Security Policy
3. ✅ **X-Frame-Options** - Clickjacking protection
4. ✅ **X-Content-Type-Options** - MIME sniffing protection
5. ✅ **Referrer-Policy** - Control referrer info
6. ✅ **Permissions-Policy** - Control browser features
7. ✅ **SRI** - Subresource Integrity
8. ✅ **Trusted Types** - XSS protection

---

### Q44. How do you implement threat modeling and risk assessment?

**Answer:**

```javascript
/**
 * Threat Modeling Service
 */
class ThreatModelingService {
  /**
   * STRIDE Threat Model
   */
  static getSTRIDEThreats() {
    return {
      Spoofing: {
        threats: [
          'Attacker impersonates legitimate user',
          'Session hijacking',
          'Man-in-the-middle attack'
        ],
        mitigations: [
          'Strong authentication (MFA)',
          'Mutual TLS (mTLS)',
          'JWT with short expiration',
          'Device fingerprinting'
        ]
      },
      
      Tampering: {
        threats: [
          'Data modification in transit',
          'Database tampering',
          'Code injection'
        ],
        mitigations: [
          'TLS encryption',
          'Database encryption',
          'Input validation',
          'Parameterized queries',
          'Integrity checks (HMAC)'
        ]
      },
      
      Repudiation: {
        threats: [
          'User denies performing action',
          'Transaction dispute'
        ],
        mitigations: [
          'Comprehensive audit logging',
          'Digital signatures',
          'Non-repudiation mechanisms',
          'Blockchain for critical transactions'
        ]
      },
      
      InformationDisclosure: {
        threats: [
          'Data breach',
          'Sensitive data exposure',
          'API information leakage'
        ],
        mitigations: [
          'Encryption at rest and in transit',
          'Access control (RBAC/ABAC)',
          'Data masking',
          'Secure error handling',
          'DLP (Data Loss Prevention)'
        ]
      },
      
      DenialOfService: {
        threats: [
          'API flooding',
          'Resource exhaustion',
          'DDoS attack'
        ],
        mitigations: [
          'Rate limiting',
          'WAF (Web Application Firewall)',
          'CDN with DDoS protection',
          'Circuit breakers',
          'Resource quotas'
        ]
      },
      
      ElevationOfPrivilege: {
        threats: [
          'Privilege escalation',
          'Unauthorized access',
          'SQL injection'
        ],
        mitigations: [
          'Principle of least privilege',
          'Input validation',
          'Secure authentication',
          'Regular permission audits'
        ]
      }
    };
  }
  
  /**
   * Risk Assessment Matrix
   */
  static calculateRiskScore(threat) {
    const likelihood = threat.likelihood; // 1-5
    const impact = threat.impact; // 1-5
    
    const riskScore = likelihood * impact;
    
    let severity;
    if (riskScore >= 20) severity = 'CRITICAL';
    else if (riskScore >= 15) severity = 'HIGH';
    else if (riskScore >= 10) severity = 'MEDIUM';
    else if (riskScore >= 5) severity = 'LOW';
    else severity = 'NEGLIGIBLE';
    
    return {
      score: riskScore,
      severity,
      likelihood,
      impact
    };
  }
  
  /**
   * ENBD Banking Threat Model
   */
  static getENBDThreatModel() {
    return [
      {
        id: 'T001',
        name: 'Account Takeover',
        description: 'Attacker gains unauthorized access to user account',
        likelihood: 4,
        impact: 5,
        mitigations: [
          'Multi-factor authentication',
          'Account lockout after failed attempts',
          'Anomaly detection',
          'Device fingerprinting'
        ],
        status: 'MITIGATED'
      },
      {
        id: 'T002',
        name: 'Transaction Fraud',
        description: 'Unauthorized fund transfers',
        likelihood: 3,
        impact: 5,
        mitigations: [
          'Transaction signing',
          'Velocity checks',
          'Fraud detection ML models',
          'Transaction limits'
        ],
        status: 'MITIGATED'
      },
      {
        id: 'T003',
        name: 'Data Breach',
        description: 'Exposure of customer PII/financial data',
        likelihood: 2,
        impact: 5,
        mitigations: [
          'Encryption at rest',
          'Field-level encryption',
          'Access controls',
          'DLP system',
          'Regular security audits'
        ],
        status: 'MITIGATED'
      },
      {
        id: 'T004',
        name: 'API Abuse',
        description: 'Excessive API calls causing service degradation',
        likelihood: 4,
        impact: 3,
        mitigations: [
          'Rate limiting',
          'API key authentication',
          'DDoS protection',
          'WAF rules'
        ],
        status: 'MITIGATED'
      },
      {
        id: 'T005',
        name: 'Insider Threat',
        description: 'Malicious employee accessing sensitive data',
        likelihood: 2,
        impact: 5,
        mitigations: [
          'Principle of least privilege',
          'Separation of duties',
          'Comprehensive audit logging',
          'Background checks',
          'Regular access reviews'
        ],
        status: 'MONITORED'
      }
    ];
  }
}

module.exports = {
  ThreatModelingService
};
```

**Threat Modeling Frameworks:**

1. ✅ **STRIDE** (Spoofing, Tampering, Repudiation, Information Disclosure, DoS, Elevation of Privilege)
2. ✅ **DREAD** (Damage, Reproducibility, Exploitability, Affected Users, Discoverability)
3. ✅ **Attack Trees**
4. ✅ **Risk Matrix** (Likelihood × Impact)

---

### Q45. How do you implement security incident response?

**Answer:**

```javascript
/**
 * Security Incident Response Service
 */
class SecurityIncidentResponseService {
  constructor() {
    this.incidents = new Map();
    this.severityLevels = ['LOW', 'MEDIUM', 'HIGH', 'CRITICAL'];
  }
  
  /**
   * Detect and create incident
   */
  async createIncident(detectionData) {
    const incident = {
      id: `INC-${Date.now()}`,
      type: detectionData.type, // 'BREACH', 'INTRUSION', 'MALWARE', 'DDoS'
      severity: this.calculateSeverity(detectionData),
      status: 'DETECTED',
      detectedAt: new Date(),
      detectedBy: detectionData.source,
      affectedSystems: detectionData.systems,
      indicators: detectionData.indicators,
      timeline: [
        {
          timestamp: new Date(),
          event: 'INCIDENT_DETECTED',
          details: detectionData
        }
      ]
    };
    
    this.incidents.set(incident.id, incident);
    
    // Immediate actions
    await this.handleIncident(incident);
    
    return incident;
  }
  
  /**
   * Incident Response Process (NIST Framework)
   */
  async handleIncident(incident) {
    // 1. PREPARATION - Already done (have response plan)
    
    // 2. DETECTION & ANALYSIS
    await this.analyzeIncident(incident);
    
    // 3. CONTAINMENT
    if (incident.severity === 'CRITICAL' || incident.severity === 'HIGH') {
      await this.containThreat(incident);
    }
    
    // 4. ERADICATION
    await this.eradicateThreat(incident);
    
    // 5. RECOVERY
    await this.recoverSystems(incident);
    
    // 6. POST-INCIDENT ACTIVITY
    await this.documentLessonsLearned(incident);
  }
  
  /**
   * Analyze incident
   */
  async analyzeIncident(incident) {
    // Gather forensic data
    const forensics = await this.collectForensics(incident);
    
    // Determine scope
    const scope = await this.determineScope(incident);
    
    // Identify indicators of compromise (IOCs)
    const iocs = await this.identifyIOCs(incident);
    
    incident.analysis = {
      forensics,
      scope,
      iocs,
      analyzedAt: new Date()
    };
    
    // Update incident
    this.addTimelineEvent(incident, 'ANALYSIS_COMPLETED', {
      forensics,
      scope,
      iocs
    });
    
    // Notify stakeholders
    await this.notifyStakeholders(incident);
  }
  
  /**
   * Contain threat
   */
  async containThreat(incident) {
    const actions = [];
    
    switch (incident.type) {
      case 'BREACH':
        // Isolate affected systems
        actions.push(await this.isolateSystem(incident.affectedSystems));
        
        // Revoke compromised credentials
        actions.push(await this.revokeCredentials(incident.indicators.compromisedUsers));
        
        // Block malicious IPs
        actions.push(await this.blockIPs(incident.indicators.maliciousIPs));
        break;
      
      case 'DDoS':
        // Enable DDoS mitigation
        actions.push(await this.enableDDoSMitigation());
        
        // Rate limiting
        actions.push(await this.enforceStrictRateLimits());
        break;
      
      case 'MALWARE':
        // Quarantine infected systems
        actions.push(await this.quarantineSystems(incident.affectedSystems));
        
        // Block malware domains
        actions.push(await this.blockDomains(incident.indicators.maliciousDomains));
        break;
    }
    
    this.addTimelineEvent(incident, 'THREAT_CONTAINED', { actions });
    incident.status = 'CONTAINED';
  }
  
  /**
   * Eradicate threat
   */
  async eradicateThreat(incident) {
    const actions = [];
    
    // Remove malware
    if (incident.type === 'MALWARE') {
      actions.push(await this.removeMalware(incident.affectedSystems));
    }
    
    // Patch vulnerabilities
    actions.push(await this.patchVulnerabilities(incident.indicators.exploitedVulnerabilities));
    
    // Reset compromised accounts
    if (incident.indicators.compromisedUsers) {
      actions.push(await this.resetAccounts(incident.indicators.compromisedUsers));
    }
    
    this.addTimelineEvent(incident, 'THREAT_ERADICATED', { actions });
    incident.status = 'ERADICATED';
  }
  
  /**
   * Recover systems
   */
  async recoverSystems(incident) {
    const actions = [];
    
    // Restore from backups if needed
    if (incident.requiresRestoration) {
      actions.push(await this.restoreFromBackup(incident.affectedSystems));
    }
    
    // Restore services
    actions.push(await this.restoreServices(incident.affectedSystems));
    
    // Monitor for reinfection
    actions.push(await this.enhanceMonitoring(incident));
    
    // Verify system integrity
    actions.push(await this.verifyIntegrity(incident.affectedSystems));
    
    this.addTimelineEvent(incident, 'SYSTEMS_RECOVERED', { actions });
    incident.status = 'RECOVERED';
  }
  
  /**
   * Post-incident review
   */
  async documentLessonsLearned(incident) {
    const report = {
      incidentId: incident.id,
      summary: this.generateSummary(incident),
      timeline: incident.timeline,
      rootCause: await this.determineRootCause(incident),
      impactAssessment: await this.assessImpact(incident),
      responseEffectiveness: this.evaluateResponse(incident),
      lessonsLearned: await this.identifyLessons(incident),
      recommendations: await this.generateRecommendations(incident),
      generatedAt: new Date()
    };
    
    // Share with team
    await this.sharePostIncidentReport(report);
    
    // Update security controls
    await this.updateSecurityControls(report.recommendations);
    
    incident.status = 'CLOSED';
    this.addTimelineEvent(incident, 'INCIDENT_CLOSED', { report });
  }
  
  /**
   * Notify stakeholders
   */
  async notifyStakeholders(incident) {
    const notifications = [];
    
    // Critical/High severity: Immediate notification
    if (['CRITICAL', 'HIGH'].includes(incident.severity)) {
      notifications.push(
        this.notifySecurityTeam(incident),
        this.notifyManagement(incident),
        this.notifyCISO(incident)
      );
    }
    
    // Data breach: Notify DPO and regulators
    if (incident.type === 'BREACH' && incident.containsPII) {
      notifications.push(
        this.notifyDPO(incident),
        this.notifyRegulators(incident)
      );
    }
    
    await Promise.all(notifications);
  }
  
  calculateSeverity(data) {
    let score = 0;
    
    if (data.containsPII) score += 3;
    if (data.affectedUsersCount > 1000) score += 3;
    if (data.financialImpact > 100000) score += 2;
    if (data.systemsDown) score += 2;
    
    if (score >= 8) return 'CRITICAL';
    if (score >= 6) return 'HIGH';
    if (score >= 4) return 'MEDIUM';
    return 'LOW';
  }
  
  addTimelineEvent(incident, event, details) {
    incident.timeline.push({
      timestamp: new Date(),
      event,
      details
    });
  }
  
  async collectForensics(incident) {
    return {};
  }
  
  async determineScope(incident) {
    return {};
  }
  
  async identifyIOCs(incident) {
    return {};
  }
  
  async isolateSystem(systems) {
    return 'Systems isolated';
  }
  
  async revokeCredentials(users) {
    return 'Credentials revoked';
  }
  
  async blockIPs(ips) {
    return 'IPs blocked';
  }
  
  async enableDDoSMitigation() {
    return 'DDoS mitigation enabled';
  }
  
  async enforceStrictRateLimits() {
    return 'Strict rate limits enforced';
  }
  
  async quarantineSystems(systems) {
    return 'Systems quarantined';
  }
  
  async blockDomains(domains) {
    return 'Domains blocked';
  }
  
  async removeMalware(systems) {
    return 'Malware removed';
  }
  
  async patchVulnerabilities(vulnerabilities) {
    return 'Vulnerabilities patched';
  }
  
  async resetAccounts(users) {
    return 'Accounts reset';
  }
  
  async restoreFromBackup(systems) {
    return 'Systems restored from backup';
  }
  
  async restoreServices(systems) {
    return 'Services restored';
  }
  
  async enhanceMonitoring(incident) {
    return 'Enhanced monitoring enabled';
  }
  
  async verifyIntegrity(systems) {
    return 'Integrity verified';
  }
  
  generateSummary(incident) {
    return 'Incident summary';
  }
  
  async determineRootCause(incident) {
    return 'Root cause analysis';
  }
  
  async assessImpact(incident) {
    return {};
  }
  
  evaluateResponse(incident) {
    return {};
  }
  
  async identifyLessons(incident) {
    return [];
  }
  
  async generateRecommendations(incident) {
    return [];
  }
  
  async sharePostIncidentReport(report) {
    // Share report
  }
  
  async updateSecurityControls(recommendations) {
    // Update controls
  }
  
  async notifySecurityTeam(incident) {
    // Notify team
  }
  
  async notifyManagement(incident) {
    // Notify management
  }
  
  async notifyCISO(incident) {
    // Notify CISO
  }
  
  async notifyDPO(incident) {
    // Notify DPO
  }
  
  async notifyRegulators(incident) {
    // Notify regulators within 72 hours (GDPR)
  }
}

module.exports = {
  SecurityIncidentResponseService
};
```

**Incident Response Process (NIST):**

1. ✅ **Preparation** - Response plan, tools, training
2. ✅ **Detection & Analysis** - Identify and analyze incidents
3. ✅ **Containment** - Isolate and limit damage
4. ✅ **Eradication** - Remove threat
5. ✅ **Recovery** - Restore systems
6. ✅ **Post-Incident Activity** - Lessons learned, improve controls

---

**Summary Q41-Q45:**
- GDPR compliance (consent, right to erasure, data portability) ✅
- Secure logging (mask PII, separate audit logs) ✅
- Browser security headers (HSTS, CSP, SRI) ✅
- Threat modeling (STRIDE, risk assessment) ✅
- Incident response (NIST framework, containment, recovery) ✅

Continuing with final security questions...
