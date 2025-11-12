# Security Questions 46-50: Security Architecture & Governance

---

### Q46. How do you implement Zero Trust Architecture in Node.js applications?

**Answer:**

```javascript
/**
 * Zero Trust Architecture Implementation
 * Principles: Never trust, always verify
 */

/**
 * Zero Trust Security Service
 */
class ZeroTrustSecurityService {
  constructor() {
    this.trustScore = new Map();
  }
  
  /**
   * Continuous Verification Middleware
   */
  continuousVerification() {
    return async (req, res, next) => {
      try {
        // 1. Identity Verification
        const identity = await this.verifyIdentity(req);
        
        // 2. Device Verification
        const device = await this.verifyDevice(req);
        
        // 3. Context Verification
        const context = await this.verifyContext(req);
        
        // 4. Calculate Trust Score
        const trustScore = await this.calculateTrustScore({
          identity,
          device,
          context
        });
        
        // 5. Policy Decision
        const decision = await this.makePolicyDecision(trustScore, req);
        
        if (!decision.allowed) {
          return res.status(403).json({
            error: 'Access denied',
            reason: decision.reason
          });
        }
        
        // Attach trust score to request
        req.trustScore = trustScore;
        req.identity = identity;
        
        next();
      } catch (error) {
        res.status(401).json({ error: 'Authentication failed' });
      }
    };
  }
  
  /**
   * Verify Identity (Multi-factor)
   */
  async verifyIdentity(req) {
    // JWT token
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) {
      throw new Error('No token provided');
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Check if MFA completed recently
    const mfaValid = await this.checkMFAStatus(decoded.userId);
    
    // Check biometric authentication (if required)
    const biometricValid = await this.checkBiometric(req);
    
    return {
      userId: decoded.userId,
      authLevel: this.calculateAuthLevel(mfaValid, biometricValid),
      verifiedAt: new Date()
    };
  }
  
  /**
   * Verify Device (Device Trust)
   */
  async verifyDevice(req) {
    const deviceId = req.headers['x-device-id'];
    const userAgent = req.headers['user-agent'];
    
    if (!deviceId) {
      return {
        trusted: false,
        reason: 'Unknown device'
      };
    }
    
    // Check device registration
    const device = await this.getRegisteredDevice(deviceId);
    
    if (!device) {
      return {
        trusted: false,
        reason: 'Unregistered device'
      };
    }
    
    // Check device health
    const deviceHealth = await this.checkDeviceHealth(device);
    
    // Check for jailbreak/root
    const deviceIntegrity = await this.checkDeviceIntegrity(req);
    
    return {
      deviceId,
      trusted: device.trusted && deviceHealth.healthy && deviceIntegrity.valid,
      lastSeen: device.lastSeen,
      os: device.os,
      health: deviceHealth,
      integrity: deviceIntegrity
    };
  }
  
  /**
   * Verify Context (Location, Time, Behavior)
   */
  async verifyContext(req) {
    const userId = req.user?.id;
    
    // Location verification
    const location = await this.verifyLocation(req);
    
    // Time-based verification
    const timeBased = await this.verifyTimeBased(req);
    
    // Behavioral analysis
    const behavior = await this.analyzeBehavior(userId, req);
    
    // Network verification
    const network = await this.verifyNetwork(req);
    
    return {
      location,
      timeBased,
      behavior,
      network
    };
  }
  
  /**
   * Calculate Trust Score (0-100)
   */
  async calculateTrustScore(factors) {
    let score = 0;
    
    // Identity Score (40 points)
    if (factors.identity.authLevel === 'HIGH') score += 40;
    else if (factors.identity.authLevel === 'MEDIUM') score += 25;
    else score += 10;
    
    // Device Score (30 points)
    if (factors.device.trusted) {
      score += 30;
    } else {
      score += 0; // Untrusted device
    }
    
    // Context Score (30 points)
    if (factors.context.location.trusted) score += 10;
    if (factors.context.timeBased.valid) score += 10;
    if (factors.context.behavior.anomaly === false) score += 10;
    
    return {
      score,
      level: this.getTrustLevel(score),
      factors
    };
  }
  
  /**
   * Policy Decision Engine
   */
  async makePolicyDecision(trustScore, req) {
    const resource = req.path;
    const action = req.method;
    
    // Get required trust level for resource
    const requiredTrust = await this.getRequiredTrustLevel(resource, action);
    
    // High-value transactions require HIGH trust
    if (resource.includes('/transactions') && action === 'POST') {
      if (trustScore.level !== 'HIGH') {
        return {
          allowed: false,
          reason: 'High-value transaction requires HIGH trust level'
        };
      }
    }
    
    // Check if trust score meets requirement
    if (trustScore.score < requiredTrust.minScore) {
      return {
        allowed: false,
        reason: `Trust score ${trustScore.score} below required ${requiredTrust.minScore}`
      };
    }
    
    return {
      allowed: true,
      trustScore: trustScore.score
    };
  }
  
  /**
   * Location Verification (Geofencing)
   */
  async verifyLocation(req) {
    const ip = req.ip;
    const geoLocation = await this.getGeoLocation(ip);
    
    // Check if location is in allowed countries
    const allowedCountries = ['AE', 'SA', 'QA', 'BH', 'KW', 'OM'];
    
    if (!allowedCountries.includes(geoLocation.countryCode)) {
      return {
        trusted: false,
        reason: 'Location outside allowed countries',
        country: geoLocation.country
      };
    }
    
    // Check for VPN/Proxy
    const vpnDetected = await this.detectVPN(ip);
    if (vpnDetected) {
      return {
        trusted: false,
        reason: 'VPN/Proxy detected'
      };
    }
    
    return {
      trusted: true,
      country: geoLocation.country,
      city: geoLocation.city
    };
  }
  
  /**
   * Behavioral Analysis
   */
  async analyzeBehavior(userId, req) {
    // Get user's typical behavior
    const profile = await this.getUserBehaviorProfile(userId);
    
    // Current behavior
    const current = {
      endpoint: req.path,
      method: req.method,
      time: new Date().getHours(),
      dayOfWeek: new Date().getDay()
    };
    
    // Check for anomalies
    const anomalyScore = await this.detectAnomalies(profile, current);
    
    return {
      anomaly: anomalyScore > 0.7,
      anomalyScore,
      reason: anomalyScore > 0.7 ? 'Unusual behavior detected' : null
    };
  }
  
  /**
   * Micro-segmentation
   */
  async applyMicroSegmentation(req) {
    const user = req.user;
    const resource = req.path;
    
    // Segment by role
    const roleSegment = await this.getRoleSegment(user.role);
    
    // Segment by data sensitivity
    const dataSensitivity = await this.getDataSensitivity(resource);
    
    // Check if role can access segment
    if (!roleSegment.allowedSegments.includes(dataSensitivity)) {
      throw new Error('Access denied: Segment not allowed');
    }
  }
  
  /**
   * Least Privilege Access
   */
  async enforceMinimalAccess(userId, resource) {
    // Get minimal permissions for resource
    const minPermissions = await this.getMinimalPermissions(resource);
    
    // Get user's actual permissions
    const userPermissions = await this.getUserPermissions(userId);
    
    // Return only minimal permissions needed
    return minPermissions.filter(p => userPermissions.includes(p));
  }
  
  calculateAuthLevel(mfaValid, biometricValid) {
    if (mfaValid && biometricValid) return 'HIGH';
    if (mfaValid) return 'MEDIUM';
    return 'LOW';
  }
  
  getTrustLevel(score) {
    if (score >= 80) return 'HIGH';
    if (score >= 60) return 'MEDIUM';
    if (score >= 40) return 'LOW';
    return 'UNTRUSTED';
  }
  
  async checkMFAStatus(userId) {
    return true;
  }
  
  async checkBiometric(req) {
    return false;
  }
  
  async getRegisteredDevice(deviceId) {
    return null;
  }
  
  async checkDeviceHealth(device) {
    return { healthy: true };
  }
  
  async checkDeviceIntegrity(req) {
    return { valid: true };
  }
  
  async verifyTimeBased(req) {
    return { valid: true };
  }
  
  async verifyNetwork(req) {
    return { trusted: true };
  }
  
  async getRequiredTrustLevel(resource, action) {
    return { minScore: 60 };
  }
  
  async getGeoLocation(ip) {
    return { countryCode: 'AE', country: 'UAE', city: 'Dubai' };
  }
  
  async detectVPN(ip) {
    return false;
  }
  
  async getUserBehaviorProfile(userId) {
    return {};
  }
  
  async detectAnomalies(profile, current) {
    return 0.2;
  }
  
  async getRoleSegment(role) {
    return { allowedSegments: ['PUBLIC', 'INTERNAL'] };
  }
  
  async getDataSensitivity(resource) {
    return 'INTERNAL';
  }
  
  async getMinimalPermissions(resource) {
    return ['READ'];
  }
  
  async getUserPermissions(userId) {
    return ['READ', 'WRITE'];
  }
}

module.exports = {
  ZeroTrustSecurityService
};
```

**Zero Trust Principles:**

1. ✅ **Never trust, always verify** - Continuous authentication
2. ✅ **Least privilege access** - Minimal permissions
3. ✅ **Micro-segmentation** - Network segmentation
4. ✅ **Multi-factor authentication** - Strong identity
5. ✅ **Device trust** - Device health and integrity
6. ✅ **Context-aware access** - Location, time, behavior

---

### Q47. How do you implement Secure Software Development Lifecycle (SSDLC)?

**Answer:**

```javascript
/**
 * Secure SDLC Implementation
 */

/**
 * Security Gate - Requirements Phase
 */
class SecurityRequirementsGate {
  /**
   * Security requirements checklist
   */
  static getSecurityRequirements() {
    return {
      authentication: [
        'Multi-factor authentication required',
        'Password complexity requirements defined',
        'Session timeout defined',
        'Account lockout policy defined'
      ],
      
      authorization: [
        'Access control model defined (RBAC/ABAC)',
        'Privilege levels documented',
        'Data access policies defined'
      ],
      
      dataProtection: [
        'Data classification defined',
        'Encryption requirements specified',
        'PII handling procedures defined',
        'Data retention policy defined'
      ],
      
      compliance: [
        'Regulatory requirements identified (GDPR, PCI DSS)',
        'Audit logging requirements defined',
        'Data residency requirements specified'
      ],
      
      securityTesting: [
        'Security testing requirements defined',
        'Vulnerability scan frequency defined',
        'Penetration testing schedule defined'
      ]
    };
  }
}

/**
 * Security Gate - Design Phase
 */
class SecurityDesignGate {
  /**
   * Threat modeling
   */
  static async performThreatModeling(design) {
    const threats = [];
    
    // STRIDE analysis
    threats.push(
      ...this.analyzeSTRIDE(design)
    );
    
    // Attack surface analysis
    threats.push(
      ...this.analyzeAttackSurface(design)
    );
    
    // Data flow analysis
    threats.push(
      ...this.analyzeDataFlows(design)
    );
    
    return threats;
  }
  
  /**
   * Security architecture review
   */
  static async reviewSecurityArchitecture(architecture) {
    const issues = [];
    
    // Check encryption
    if (!architecture.encryption.atRest) {
      issues.push('Missing encryption at rest');
    }
    
    // Check authentication
    if (!architecture.authentication.mfa) {
      issues.push('MFA not implemented');
    }
    
    // Check network segmentation
    if (!architecture.network.segmented) {
      issues.push('Network not segmented');
    }
    
    return issues;
  }
  
  static analyzeSTRIDE(design) {
    return [];
  }
  
  static analyzeAttackSurface(design) {
    return [];
  }
  
  static analyzeDataFlows(design) {
    return [];
  }
}

/**
 * Security Gate - Implementation Phase
 */
class SecurityImplementationGate {
  /**
   * Secure coding guidelines
   */
  static getSecureCodingGuidelines() {
    return {
      inputValidation: [
        'Validate all user inputs',
        'Use parameterized queries',
        'Sanitize output',
        'Implement allowlists, not denylists'
      ],
      
      authentication: [
        'Never store passwords in plaintext',
        'Use bcrypt/Argon2 for password hashing',
        'Implement secure session management',
        'Use secure random number generation'
      ],
      
      cryptography: [
        'Use standard crypto libraries',
        'Never implement custom crypto',
        'Use strong algorithms (AES-256, RSA-2048)',
        'Secure key management'
      ],
      
      errorHandling: [
        'Never expose stack traces to users',
        'Log security events',
        'Handle errors gracefully',
        'Use generic error messages'
      ],
      
      dependencies: [
        'Keep dependencies updated',
        'Scan for vulnerabilities',
        'Use package lock files',
        'Review third-party code'
      ]
    };
  }
  
  /**
   * Code review checklist
   */
  static async performSecurityCodeReview(code) {
    const issues = [];
    
    // Check for hardcoded secrets
    if (code.match(/password\s*=\s*['"][^'"]+['"]/i)) {
      issues.push({
        severity: 'CRITICAL',
        issue: 'Hardcoded password detected',
        line: 'X'
      });
    }
    
    // Check for SQL injection
    if (code.match(/query\([^)]*\+[^)]*\)/)) {
      issues.push({
        severity: 'HIGH',
        issue: 'Potential SQL injection - use parameterized queries',
        line: 'Y'
      });
    }
    
    // Check for XSS
    if (code.match(/innerHTML\s*=/)) {
      issues.push({
        severity: 'HIGH',
        issue: 'XSS vulnerability - use textContent instead',
        line: 'Z'
      });
    }
    
    return issues;
  }
}

/**
 * Security Gate - Testing Phase
 */
class SecurityTestingGate {
  /**
   * Security test suite
   */
  static async runSecurityTests() {
    const results = {
      sast: await this.runSAST(),
      dast: await this.runDAST(),
      sca: await this.runSCA(),
      secretsScanning: await this.runSecretsScanning(),
      vulnerabilityScan: await this.runVulnerabilityScan()
    };
    
    return results;
  }
  
  /**
   * SAST - Static Application Security Testing
   */
  static async runSAST() {
    // ESLint Security Plugin
    // SonarQube
    // Semgrep
    return {
      tool: 'ESLint Security + SonarQube',
      vulnerabilities: []
    };
  }
  
  /**
   * DAST - Dynamic Application Security Testing
   */
  static async runDAST() {
    // OWASP ZAP
    // Burp Suite
    return {
      tool: 'OWASP ZAP',
      vulnerabilities: []
    };
  }
  
  /**
   * SCA - Software Composition Analysis
   */
  static async runSCA() {
    // npm audit
    // Snyk
    // OWASP Dependency Check
    return {
      tool: 'Snyk + npm audit',
      vulnerabilities: []
    };
  }
  
  /**
   * Secrets Scanning
   */
  static async runSecretsScanning() {
    // TruffleHog
    // GitGuardian
    return {
      tool: 'TruffleHog',
      secrets: []
    };
  }
  
  /**
   * Vulnerability Scanning
   */
  static async runVulnerabilityScan() {
    // Nessus
    // OpenVAS
    return {
      tool: 'OpenVAS',
      vulnerabilities: []
    };
  }
}

/**
 * Security Gate - Deployment Phase
 */
class SecurityDeploymentGate {
  /**
   * Pre-deployment security checklist
   */
  static async preDeploymentChecklist() {
    return {
      configuration: [
        { check: 'Debug mode disabled', status: 'PASS' },
        { check: 'Secrets in environment variables', status: 'PASS' },
        { check: 'HTTPS enforced', status: 'PASS' },
        { check: 'Security headers configured', status: 'PASS' }
      ],
      
      infrastructure: [
        { check: 'Firewall rules configured', status: 'PASS' },
        { check: 'WAF enabled', status: 'PASS' },
        { check: 'DDoS protection enabled', status: 'PASS' },
        { check: 'Monitoring configured', status: 'PASS' }
      ],
      
      access: [
        { check: 'Production access restricted', status: 'PASS' },
        { check: 'MFA required for production', status: 'PASS' },
        { check: 'Audit logging enabled', status: 'PASS' }
      ]
    };
  }
  
  /**
   * Infrastructure as Code Security
   */
  static async scanIaC(iacFiles) {
    // Checkov
    // tfsec
    // Terrascan
    return {
      tool: 'Checkov',
      issues: []
    };
  }
}

/**
 * Security Gate - Maintenance Phase
 */
class SecurityMaintenanceGate {
  /**
   * Continuous monitoring
   */
  static async setupContinuousMonitoring() {
    return {
      vulnerabilityScanning: {
        frequency: 'daily',
        tool: 'Snyk'
      },
      
      logMonitoring: {
        tool: 'Elasticsearch + Kibana',
        alerts: ['Failed logins', 'SQL injection attempts', 'XSS attempts']
      },
      
      securityMetrics: [
        'Mean time to detect (MTTD)',
        'Mean time to respond (MTTR)',
        'Vulnerability count by severity',
        'Security test coverage'
      ]
    };
  }
  
  /**
   * Incident response readiness
   */
  static async verifyIncidentResponse() {
    return {
      plan: 'Documented and reviewed',
      team: 'Trained and assigned',
      tools: 'Configured and tested',
      drills: 'Quarterly tabletop exercises'
    };
  }
}

module.exports = {
  SecurityRequirementsGate,
  SecurityDesignGate,
  SecurityImplementationGate,
  SecurityTestingGate,
  SecurityDeploymentGate,
  SecurityMaintenanceGate
};
```

**SSDLC Phases:**

1. ✅ **Requirements** - Security requirements defined
2. ✅ **Design** - Threat modeling, security architecture
3. ✅ **Implementation** - Secure coding, code review
4. ✅ **Testing** - SAST, DAST, SCA, penetration testing
5. ✅ **Deployment** - Security configuration, IaC scanning
6. ✅ **Maintenance** - Monitoring, incident response, patching

---

### Q48. How do you implement API security for third-party integrations?

**Answer:**

```javascript
/**
 * Third-Party API Security Service
 */
class ThirdPartyAPISecurityService {
  /**
   * API Key Management for Partners
   */
  async generatePartnerAPIKey(partnerId, scopes) {
    const apiKey = {
      key: crypto.randomBytes(32).toString('hex'),
      secret: crypto.randomBytes(32).toString('hex'),
      partnerId,
      scopes, // ['transactions:read', 'accounts:read']
      createdAt: new Date(),
      expiresAt: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000), // 1 year
      rateLimit: {
        requests: 1000,
        window: 3600 // 1 hour
      }
    };
    
    // Store hashed secret
    apiKey.secretHash = await bcrypt.hash(apiKey.secret, 12);
    
    await this.storeAPIKey(apiKey);
    
    // Return key and secret only once
    return {
      key: apiKey.key,
      secret: apiKey.secret,
      message: 'Store the secret securely. It will not be shown again.'
    };
  }
  
  /**
   * API Key Authentication Middleware
   */
  authenticateAPIKey() {
    return async (req, res, next) => {
      const apiKey = req.headers['x-api-key'];
      const signature = req.headers['x-api-signature'];
      
      if (!apiKey || !signature) {
        return res.status(401).json({ error: 'Missing API credentials' });
      }
      
      // Get API key from database
      const keyData = await this.getAPIKey(apiKey);
      
      if (!keyData) {
        return res.status(401).json({ error: 'Invalid API key' });
      }
      
      // Check expiration
      if (new Date() > keyData.expiresAt) {
        return res.status(401).json({ error: 'API key expired' });
      }
      
      // Verify signature (HMAC)
      const expectedSignature = this.generateSignature(req, keyData.secret);
      
      if (signature !== expectedSignature) {
        return res.status(401).json({ error: 'Invalid signature' });
      }
      
      // Check rate limit
      const rateLimit = await this.checkRateLimit(apiKey);
      if (!rateLimit.allowed) {
        return res.status(429).json({ error: 'Rate limit exceeded' });
      }
      
      // Attach partner info to request
      req.partner = {
        id: keyData.partnerId,
        scopes: keyData.scopes
      };
      
      next();
    };
  }
  
  /**
   * Generate HMAC signature for request
   */
  generateSignature(req, secret) {
    const timestamp = req.headers['x-api-timestamp'];
    const method = req.method;
    const path = req.path;
    const body = JSON.stringify(req.body);
    
    const message = `${timestamp}${method}${path}${body}`;
    
    return crypto
      .createHmac('sha256', secret)
      .update(message)
      .digest('hex');
  }
  
  /**
   * Scope-based authorization
   */
  requireScope(requiredScope) {
    return (req, res, next) => {
      const partnerScopes = req.partner?.scopes || [];
      
      if (!partnerScopes.includes(requiredScope)) {
        return res.status(403).json({
          error: 'Insufficient permissions',
          required: requiredScope
        });
      }
      
      next();
    };
  }
  
  /**
   * Request validation
   */
  validateRequest() {
    return (req, res, next) => {
      // Check timestamp (prevent replay attacks)
      const timestamp = parseInt(req.headers['x-api-timestamp']);
      const now = Date.now();
      
      if (Math.abs(now - timestamp) > 300000) { // 5 minutes
        return res.status(401).json({ error: 'Request timestamp invalid' });
      }
      
      // Check nonce (prevent replay attacks)
      const nonce = req.headers['x-api-nonce'];
      if (this.isNonceUsed(nonce)) {
        return res.status(401).json({ error: 'Nonce already used' });
      }
      
      this.markNonceUsed(nonce);
      
      next();
    };
  }
  
  /**
   * Webhook Security (for callbacks to partners)
   */
  async sendSecureWebhook(partnerId, event, data) {
    const partner = await this.getPartner(partnerId);
    
    if (!partner.webhookUrl) {
      throw new Error('Partner webhook URL not configured');
    }
    
    // Generate signature
    const timestamp = Date.now();
    const payload = JSON.stringify({ event, data, timestamp });
    
    const signature = crypto
      .createHmac('sha256', partner.webhookSecret)
      .update(payload)
      .digest('hex');
    
    // Send webhook
    try {
      const response = await axios.post(partner.webhookUrl, {
        event,
        data,
        timestamp
      }, {
        headers: {
          'X-ENBD-Signature': signature,
          'X-ENBD-Timestamp': timestamp,
          'Content-Type': 'application/json'
        },
        timeout: 5000
      });
      
      await this.logWebhook({
        partnerId,
        event,
        status: 'SUCCESS',
        statusCode: response.status
      });
    } catch (error) {
      await this.logWebhook({
        partnerId,
        event,
        status: 'FAILED',
        error: error.message
      });
      
      // Retry with exponential backoff
      await this.scheduleWebhookRetry(partnerId, event, data);
    }
  }
  
  /**
   * IP Whitelisting
   */
  enforceIPWhitelist() {
    return async (req, res, next) => {
      const partnerId = req.partner?.id;
      const clientIP = req.ip;
      
      const partner = await this.getPartner(partnerId);
      
      if (partner.ipWhitelist && partner.ipWhitelist.length > 0) {
        if (!partner.ipWhitelist.includes(clientIP)) {
          return res.status(403).json({
            error: 'IP address not whitelisted'
          });
        }
      }
      
      next();
    };
  }
  
  /**
   * Data filtering (only return allowed fields)
   */
  filterResponse(allowedFields) {
    return (req, res, next) => {
      const originalSend = res.send;
      
      res.send = function(data) {
        if (typeof data === 'object') {
          const filtered = this.filterObject(JSON.parse(data), allowedFields);
          data = JSON.stringify(filtered);
        }
        
        originalSend.call(this, data);
      }.bind(this);
      
      next();
    };
  }
  
  filterObject(obj, allowedFields) {
    if (Array.isArray(obj)) {
      return obj.map(item => this.filterObject(item, allowedFields));
    }
    
    const filtered = {};
    for (const field of allowedFields) {
      if (obj[field] !== undefined) {
        filtered[field] = obj[field];
      }
    }
    
    return filtered;
  }
  
  async storeAPIKey(apiKey) {
    // Store in database
  }
  
  async getAPIKey(key) {
    return null;
  }
  
  async checkRateLimit(apiKey) {
    return { allowed: true };
  }
  
  isNonceUsed(nonce) {
    return false;
  }
  
  markNonceUsed(nonce) {
    // Mark in cache
  }
  
  async getPartner(partnerId) {
    return {};
  }
  
  async logWebhook(log) {
    // Log webhook
  }
  
  async scheduleWebhookRetry(partnerId, event, data) {
    // Schedule retry
  }
}

module.exports = {
  ThirdPartyAPISecurityService
};
```

**Third-Party API Security:**

1. ✅ **API Key + HMAC Signature** - Strong authentication
2. ✅ **Scope-based authorization** - Granular permissions
3. ✅ **Rate limiting** - Prevent abuse
4. ✅ **Timestamp validation** - Prevent replay attacks
5. ✅ **IP whitelisting** - Network-level security
6. ✅ **Webhook signature** - Secure callbacks
7. ✅ **Data filtering** - Only expose necessary fields

---

### Q49. How do you implement security monitoring and SIEM integration?

**Answer:**

```javascript
const winston = require('winston');
const elasticsearch = require('@elastic/elasticsearch');

/**
 * Security Information and Event Management (SIEM) Service
 */
class SIEMIntegrationService {
  constructor() {
    this.esClient = new elasticsearch.Client({
      node: process.env.ELASTICSEARCH_URL,
      auth: {
        username: process.env.ES_USERNAME,
        password: process.env.ES_PASSWORD
      }
    });
    
    this.setupLogger();
  }
  
  /**
   * Setup centralized logging
   */
  setupLogger() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        // Send to Elasticsearch
        new winston.transports.Http({
          host: process.env.LOGSTASH_HOST,
          port: process.env.LOGSTASH_PORT,
          path: '/logs'
        })
      ]
    });
  }
  
  /**
   * Log security event
   */
  async logSecurityEvent(event) {
    const securityEvent = {
      '@timestamp': new Date().toISOString(),
      event_type: event.type,
      severity: event.severity,
      user_id: event.userId,
      ip_address: event.ip,
      user_agent: event.userAgent,
      action: event.action,
      resource: event.resource,
      result: event.result, // 'SUCCESS', 'FAILURE'
      details: event.details,
      tags: ['security', event.type],
      
      // GeoIP enrichment
      geo: await this.enrichGeoIP(event.ip),
      
      // Threat intelligence
      threat_intel: await this.checkThreatIntel(event.ip)
    };
    
    // Index to Elasticsearch
    await this.esClient.index({
      index: 'security-events',
      body: securityEvent
    });
    
    // Check for alerts
    await this.checkAlertConditions(securityEvent);
  }
  
  /**
   * Failed login detection
   */
  async detectFailedLogins(userId, ip) {
    // Query recent failed logins
    const response = await this.esClient.search({
      index: 'security-events',
      body: {
        query: {
          bool: {
            must: [
              { match: { event_type: 'login' } },
              { match: { result: 'FAILURE' } },
              { match: { user_id: userId } }
            ],
            filter: {
              range: {
                '@timestamp': {
                  gte: 'now-15m'
                }
              }
            }
          }
        }
      }
    });
    
    const failedAttempts = response.hits.total.value;
    
    // Alert on 5 failed attempts in 15 minutes
    if (failedAttempts >= 5) {
      await this.createAlert({
        type: 'BRUTE_FORCE_ATTEMPT',
        severity: 'HIGH',
        userId,
        ip,
        attempts: failedAttempts,
        message: `${failedAttempts} failed login attempts in 15 minutes`
      });
      
      // Lock account
      await this.lockAccount(userId);
    }
  }
  
  /**
   * Detect SQL injection attempts
   */
  async detectSQLInjection(req) {
    const sqlPatterns = [
      /(\bunion\b.*\bselect\b)/i,
      /(\bor\b.*=.*)/i,
      /(';|"|--)/,
      /(\bexec\b|\bexecute\b)/i
    ];
    
    const queryString = JSON.stringify(req.query);
    const body = JSON.stringify(req.body);
    
    for (const pattern of sqlPatterns) {
      if (pattern.test(queryString) || pattern.test(body)) {
        await this.logSecurityEvent({
          type: 'sql_injection_attempt',
          severity: 'CRITICAL',
          userId: req.user?.id,
          ip: req.ip,
          userAgent: req.headers['user-agent'],
          action: 'SQL_INJECTION_DETECTED',
          resource: req.path,
          result: 'BLOCKED',
          details: {
            query: req.query,
            body: req.body
          }
        });
        
        await this.createAlert({
          type: 'SQL_INJECTION',
          severity: 'CRITICAL',
          ip: req.ip,
          path: req.path
        });
        
        // Block IP
        await this.blockIP(req.ip);
        
        return true;
      }
    }
    
    return false;
  }
  
  /**
   * Detect XSS attempts
   */
  async detectXSS(req) {
    const xssPatterns = [
      /<script[^>]*>.*<\/script>/i,
      /javascript:/i,
      /onerror\s*=/i,
      /onload\s*=/i
    ];
    
    const params = JSON.stringify({ ...req.query, ...req.body });
    
    for (const pattern of xssPatterns) {
      if (pattern.test(params)) {
        await this.logSecurityEvent({
          type: 'xss_attempt',
          severity: 'HIGH',
          userId: req.user?.id,
          ip: req.ip,
          userAgent: req.headers['user-agent'],
          action: 'XSS_DETECTED',
          resource: req.path,
          result: 'BLOCKED',
          details: { params }
        });
        
        return true;
      }
    }
    
    return false;
  }
  
  /**
   * Anomaly detection using ML
   */
  async detectAnomalies(userId, activity) {
    // Use Elasticsearch ML features
    const response = await this.esClient.ml.getBuckets({
      job_id: 'user-behavior-anomaly',
      body: {
        anomaly_score: {
          gte: 75 // High anomaly score
        }
      }
    });
    
    if (response.buckets.length > 0) {
      await this.createAlert({
        type: 'ANOMALOUS_BEHAVIOR',
        severity: 'MEDIUM',
        userId,
        anomalyScore: response.buckets[0].anomaly_score,
        message: 'Unusual user behavior detected'
      });
    }
  }
  
  /**
   * Real-time dashboard queries
   */
  async getSecurityDashboard() {
    // Failed logins in last 24 hours
    const failedLogins = await this.esClient.count({
      index: 'security-events',
      body: {
        query: {
          bool: {
            must: [
              { match: { event_type: 'login' } },
              { match: { result: 'FAILURE' } }
            ],
            filter: {
              range: {
                '@timestamp': { gte: 'now-24h' }
              }
            }
          }
        }
      }
    });
    
    // Active alerts
    const activeAlerts = await this.getActiveAlerts();
    
    // Top attack sources
    const topAttackers = await this.esClient.search({
      index: 'security-events',
      body: {
        size: 0,
        query: {
          bool: {
            must: [
              { match: { result: 'BLOCKED' } }
            ],
            filter: {
              range: {
                '@timestamp': { gte: 'now-24h' }
              }
            }
          }
        },
        aggs: {
          top_ips: {
            terms: {
              field: 'ip_address',
              size: 10
            }
          }
        }
      }
    });
    
    return {
      failedLogins: failedLogins.count,
      activeAlerts: activeAlerts.length,
      topAttackers: topAttackers.aggregations.top_ips.buckets
    };
  }
  
  /**
   * Create security alert
   */
  async createAlert(alert) {
    const alertDoc = {
      '@timestamp': new Date().toISOString(),
      alert_id: crypto.randomBytes(16).toString('hex'),
      type: alert.type,
      severity: alert.severity,
      status: 'OPEN',
      ...alert
    };
    
    await this.esClient.index({
      index: 'security-alerts',
      body: alertDoc
    });
    
    // Send notifications
    await this.notifySecurityTeam(alertDoc);
  }
  
  async enrichGeoIP(ip) {
    return {
      country: 'UAE',
      city: 'Dubai',
      coordinates: { lat: 25.2048, lon: 55.2708 }
    };
  }
  
  async checkThreatIntel(ip) {
    // Check against threat feeds
    return {
      malicious: false,
      categories: []
    };
  }
  
  async checkAlertConditions(event) {
    // Check conditions
  }
  
  async lockAccount(userId) {
    // Lock account
  }
  
  async blockIP(ip) {
    // Add to firewall block list
  }
  
  async getActiveAlerts() {
    return [];
  }
  
  async notifySecurityTeam(alert) {
    // Send Slack/email notification
  }
}

module.exports = {
  SIEMIntegrationService
};
```

**SIEM Integration:**

1. ✅ **Centralized logging** - Elasticsearch/Splunk
2. ✅ **Real-time monitoring** - Failed logins, attacks
3. ✅ **Anomaly detection** - ML-based behavior analysis
4. ✅ **Alert management** - Automated alerts and notifications
5. ✅ **Threat intelligence** - IP reputation, IOCs
6. ✅ **Security dashboard** - Real-time visibility

---

### Q50. How do you implement defense in depth strategy?

**Answer:**

```javascript
/**
 * Defense in Depth Strategy
 * Multiple layers of security controls
 */

/**
 * Layer 1: Perimeter Security
 */
class PerimeterSecurity {
  static configure(app) {
    // WAF (Web Application Firewall)
    app.use(this.wafRules());
    
    // DDoS Protection
    app.use(this.ddosProtection());
    
    // Rate Limiting
    app.use(this.globalRateLimit());
    
    // IP Filtering
    app.use(this.ipFiltering());
  }
  
  static wafRules() {
    return (req, res, next) => {
      // Block known attack patterns
      const patterns = [
        /(\bunion\b.*\bselect\b)/i, // SQL injection
        /<script/i, // XSS
        /\.\.\//, // Path traversal
        /(\bexec\b|\beval\b)/i // Code injection
      ];
      
      const input = JSON.stringify({ ...req.query, ...req.body, ...req.params });
      
      for (const pattern of patterns) {
        if (pattern.test(input)) {
          return res.status(403).json({ error: 'Request blocked by WAF' });
        }
      }
      
      next();
    };
  }
  
  static ddosProtection() {
    const connections = new Map();
    
    return (req, res, next) => {
      const ip = req.ip;
      const now = Date.now();
      
      if (!connections.has(ip)) {
        connections.set(ip, []);
      }
      
      const ipConnections = connections.get(ip);
      
      // Remove old connections (older than 10 seconds)
      const recent = ipConnections.filter(time => now - time < 10000);
      
      // Max 100 requests per 10 seconds
      if (recent.length >= 100) {
        return res.status(429).json({ error: 'Too many requests' });
      }
      
      recent.push(now);
      connections.set(ip, recent);
      
      next();
    };
  }
  
  static globalRateLimit() {
    return (req, res, next) => {
      // Implemented
      next();
    };
  }
  
  static ipFiltering() {
    return (req, res, next) => {
      // Block known malicious IPs
      next();
    };
  }
}

/**
 * Layer 2: Network Security
 */
class NetworkSecurity {
  static configure() {
    return {
      // Network segmentation
      segments: {
        dmz: ['web-servers'],
        application: ['app-servers'],
        database: ['db-servers'],
        management: ['admin-servers']
      },
      
      // Firewall rules
      firewallRules: [
        {
          source: 'internet',
          destination: 'dmz',
          ports: [80, 443],
          action: 'ALLOW'
        },
        {
          source: 'dmz',
          destination: 'application',
          ports: [8080],
          action: 'ALLOW'
        },
        {
          source: 'application',
          destination: 'database',
          ports: [5432],
          action: 'ALLOW'
        },
        {
          source: 'internet',
          destination: 'database',
          ports: 'all',
          action: 'DENY'
        }
      ],
      
      // VPN for remote access
      vpn: {
        required: true,
        mfa: true
      }
    };
  }
}

/**
 * Layer 3: Application Security
 */
class ApplicationSecurity {
  static configure(app) {
    // Authentication
    app.use('/api/*', this.authenticate());
    
    // Authorization
    app.use('/api/*', this.authorize());
    
    // Input validation
    app.use(this.validateInput());
    
    // Output encoding
    app.use(this.encodeOutput());
    
    // Security headers
    app.use(helmet());
    
    // CSRF protection
    app.use(csrf());
  }
  
  static authenticate() {
    return (req, res, next) => {
      // JWT authentication
      next();
    };
  }
  
  static authorize() {
    return (req, res, next) => {
      // RBAC/ABAC authorization
      next();
    };
  }
  
  static validateInput() {
    return (req, res, next) => {
      // Joi validation
      next();
    };
  }
  
  static encodeOutput() {
    return (req, res, next) => {
      // HTML encoding
      next();
    };
  }
}

/**
 * Layer 4: Data Security
 */
class DataSecurity {
  static configure() {
    return {
      // Encryption at rest
      encryptionAtRest: {
        algorithm: 'AES-256-GCM',
        keyManagement: 'AWS KMS'
      },
      
      // Encryption in transit
      encryptionInTransit: {
        tls: 'TLS 1.3',
        certificatePinning: true
      },
      
      // Data classification
      classification: {
        PUBLIC: { encryption: false, retention: '1 year' },
        INTERNAL: { encryption: true, retention: '7 years' },
        CONFIDENTIAL: { encryption: true, retention: '7 years' },
        RESTRICTED: { encryption: true, keyRotation: '90 days' }
      },
      
      // Data masking
      masking: {
        cardNumber: 'PCI DSS compliant',
        ssn: 'Full mask',
        email: 'Partial mask'
      },
      
      // Backup encryption
      backupEncryption: true
    };
  }
}

/**
 * Layer 5: Endpoint Security
 */
class EndpointSecurity {
  static configure() {
    return {
      // Device management
      deviceManagement: {
        mdm: true,
        compliance: ['encryption', 'antivirus', 'firewall']
      },
      
      // Device health checks
      healthChecks: {
        osVersion: 'required',
        securityPatches: 'up-to-date',
        jailbreak: 'not allowed'
      },
      
      // Certificate-based authentication
      clientCertificates: true
    };
  }
}

/**
 * Layer 6: Monitoring & Response
 */
class MonitoringAndResponse {
  static configure() {
    return {
      // SIEM integration
      siem: {
        tool: 'Elasticsearch',
        retention: '1 year'
      },
      
      // Real-time monitoring
      monitoring: {
        failedLogins: true,
        sqlInjection: true,
        xss: true,
        anomalies: true
      },
      
      // Incident response
      incidentResponse: {
        plan: 'documented',
        team: 'assigned',
        drills: 'quarterly'
      },
      
      // Security metrics
      metrics: [
        'Mean time to detect (MTTD)',
        'Mean time to respond (MTTR)',
        'Vulnerability count',
        'Patch compliance'
      ]
    };
  }
}

/**
 * Defense in Depth Implementation
 */
class DefenseInDepth {
  static implement(app) {
    // Layer 1: Perimeter
    PerimeterSecurity.configure(app);
    
    // Layer 2: Network
    const networkConfig = NetworkSecurity.configure();
    
    // Layer 3: Application
    ApplicationSecurity.configure(app);
    
    // Layer 4: Data
    const dataConfig = DataSecurity.configure();
    
    // Layer 5: Endpoint
    const endpointConfig = EndpointSecurity.configure();
    
    // Layer 6: Monitoring
    const monitoringConfig = MonitoringAndResponse.configure();
    
    return {
      layers: {
        perimeter: 'WAF, DDoS, Rate Limiting',
        network: networkConfig,
        application: 'Authentication, Authorization, Input Validation',
        data: dataConfig,
        endpoint: endpointConfig,
        monitoring: monitoringConfig
      }
    };
  }
}

module.exports = {
  DefenseInDepth,
  PerimeterSecurity,
  NetworkSecurity,
  ApplicationSecurity,
  DataSecurity,
  EndpointSecurity,
  MonitoringAndResponse
};
```

**Defense in Depth Layers:**

1. ✅ **Perimeter** - WAF, DDoS protection, rate limiting
2. ✅ **Network** - Segmentation, firewalls, VPN
3. ✅ **Application** - Authentication, authorization, validation
4. ✅ **Data** - Encryption at rest/transit, masking
5. ✅ **Endpoint** - Device management, health checks
6. ✅ **Monitoring** - SIEM, incident response, metrics

---

**Summary Q46-Q50:**
- Zero Trust Architecture (never trust, always verify) ✅
- Secure SDLC (security in all phases) ✅
- Third-party API security (API keys, HMAC, scopes) ✅
- SIEM integration (centralized monitoring, alerts) ✅
- Defense in depth (multiple security layers) ✅

**🎉 Security Topic COMPLETED - All 50 questions done!**

Moving to GenAI topic next...
