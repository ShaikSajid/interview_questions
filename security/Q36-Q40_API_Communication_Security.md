# Security Questions 36-40: Secure Communication & API Security

---

### Q36. How do you implement API versioning and deprecation securely?

**Answer:**

```javascript
const express = require('express');

/**
 * API Versioning Service
 */
class APIVersioningService {
  constructor() {
    this.deprecatedVersions = new Map();
    this.supportedVersions = ['v1', 'v2', 'v3'];
  }
  
  /**
   * URL-based versioning
   */
  setupURLVersioning(app) {
    // Version 1 (deprecated)
    app.get('/api/v1/accounts', this.deprecationWarning('v1'), async (req, res) => {
      res.json({ version: 'v1', accounts: [] });
    });
    
    // Version 2 (current)
    app.get('/api/v2/accounts', async (req, res) => {
      res.json({ version: 'v2', accounts: [] });
    });
    
    // Version 3 (latest)
    app.get('/api/v3/accounts', async (req, res) => {
      res.json({ version: 'v3', accounts: [] });
    });
  }
  
  /**
   * Header-based versioning
   */
  getVersionFromHeader() {
    return (req, res, next) => {
      const version = req.headers['api-version'] || 
                     req.headers['accept-version'] ||
                     'v3'; // Default to latest
      
      if (!this.supportedVersions.includes(version)) {
        return res.status(400).json({
          error: 'Unsupported API version',
          supported: this.supportedVersions
        });
      }
      
      req.apiVersion = version;
      next();
    };
  }
  
  /**
   * Deprecation warning middleware
   */
  deprecationWarning(version) {
    return (req, res, next) => {
      const deprecationInfo = this.deprecatedVersions.get(version);
      
      if (deprecationInfo) {
        res.setHeader('Warning', `299 - "API version ${version} is deprecated"`);
        res.setHeader('Deprecation', 'true');
        res.setHeader('Sunset', deprecationInfo.sunsetDate.toISOString());
        res.setHeader('Link', `<${deprecationInfo.migrationGuide}>; rel="deprecation"`);
      }
      
      next();
    };
  }
  
  /**
   * Mark version as deprecated
   */
  deprecateVersion(version, sunsetDate, migrationGuide) {
    this.deprecatedVersions.set(version, {
      sunsetDate: new Date(sunsetDate),
      migrationGuide
    });
    
    // Log deprecation
    console.log(`API ${version} deprecated. Sunset: ${sunsetDate}`);
  }
  
  /**
   * Check if version is still supported
   */
  isVersionSupported(version) {
    const deprecation = this.deprecatedVersions.get(version);
    
    if (!deprecation) {
      return true; // Not deprecated
    }
    
    // Check if sunset date has passed
    return new Date() < deprecation.sunsetDate;
  }
}

/**
 * API Rate Limiting per Version
 */
class VersionedRateLimitingService {
  constructor() {
    this.limits = {
      'v1': { requests: 100, window: 3600 },  // Legacy: 100/hour
      'v2': { requests: 500, window: 3600 },  // Current: 500/hour
      'v3': { requests: 1000, window: 3600 }  // Latest: 1000/hour
    };
  }
  
  getRateLimitForVersion(version) {
    return this.limits[version] || this.limits['v3'];
  }
}

/**
 * Backward Compatibility Service
 */
class BackwardCompatibilityService {
  /**
   * Transform v1 response to v2 format
   */
  transformV1ToV2(v1Data) {
    return {
      ...v1Data,
      metadata: {
        version: 'v2',
        transformedFrom: 'v1'
      }
    };
  }
  
  /**
   * Transform v2 request to v3 format
   */
  transformV2ToV3Request(v2Request) {
    return {
      ...v2Request,
      newFields: {
        // Add new required fields with defaults
      }
    };
  }
}

module.exports = {
  APIVersioningService,
  VersionedRateLimitingService,
  BackwardCompatibilityService
};
```

**API Versioning Best Practices:**

1. ✅ **Clear deprecation warnings**
2. ✅ **Sunset HTTP header**
3. ✅ **Migration guides**
4. ✅ **Version-specific rate limits**
5. ✅ **Backward compatibility**

---

### Q37. How do you implement secure GraphQL APIs?

**Answer:**

```javascript
const { graphqlHTTP } = require('express-graphql');
const { GraphQLSchema, GraphQLObjectType, GraphQLString, GraphQLInt } = require('graphql');
const depthLimit = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

/**
 * GraphQL Security Service
 */
class GraphQLSecurityService {
  /**
   * Query depth limiting (prevent nested query attacks)
   */
  getDepthLimitRule() {
    return depthLimit(5); // Max depth of 5
  }
  
  /**
   * Query complexity limiting
   */
  getComplexityLimitRule() {
    return createComplexityLimitRule(1000, {
      onCost: (cost) => {
        console.log('Query cost:', cost);
      }
    });
  }
  
  /**
   * Disable introspection in production
   */
  disableIntrospection(schema) {
    if (process.env.NODE_ENV === 'production') {
      return {
        validationRules: [
          (context) => ({
            Field(node) {
              if (node.name.value === '__schema' || node.name.value === '__type') {
                context.reportError(
                  new Error('GraphQL introspection is disabled in production')
                );
              }
            }
          })
        ]
      };
    }
    return {};
  }
  
  /**
   * Field-level authorization
   */
  createAuthorizedField(fieldConfig, requiredRole) {
    const originalResolve = fieldConfig.resolve;
    
    fieldConfig.resolve = async (source, args, context, info) => {
      if (!context.user) {
        throw new Error('Authentication required');
      }
      
      if (context.user.role !== requiredRole) {
        throw new Error('Insufficient permissions');
      }
      
      return originalResolve(source, args, context, info);
    };
    
    return fieldConfig;
  }
  
  /**
   * Query timeout
   */
  setupQueryTimeout(timeoutMs = 10000) {
    return {
      extensions: {
        timeout: timeoutMs
      }
    };
  }
  
  /**
   * Input sanitization
   */
  sanitizeInput(input) {
    if (typeof input === 'string') {
      return input.trim().substring(0, 1000);
    }
    
    if (typeof input === 'object') {
      const sanitized = {};
      for (const [key, value] of Object.entries(input)) {
        sanitized[key] = this.sanitizeInput(value);
      }
      return sanitized;
    }
    
    return input;
  }
}

/**
 * Secure GraphQL Schema
 */
class SecureGraphQLSchema {
  constructor() {
    this.securityService = new GraphQLSecurityService();
  }
  
  createSchema() {
    const UserType = new GraphQLObjectType({
      name: 'User',
      fields: {
        id: { type: GraphQLString },
        email: { type: GraphQLString },
        name: { type: GraphQLString },
        
        // Sensitive field with authorization
        ssn: this.securityService.createAuthorizedField({
          type: GraphQLString,
          resolve: (user) => user.ssn
        }, 'admin')
      }
    });
    
    const QueryType = new GraphQLObjectType({
      name: 'Query',
      fields: {
        user: {
          type: UserType,
          args: {
            id: { type: GraphQLString }
          },
          resolve: async (parent, args, context) => {
            // Authentication check
            if (!context.user) {
              throw new Error('Authentication required');
            }
            
            // Sanitize input
            const sanitizedId = this.securityService.sanitizeInput(args.id);
            
            // Authorization check
            if (context.user.id !== sanitizedId && context.user.role !== 'admin') {
              throw new Error('Unauthorized');
            }
            
            return await this.getUser(sanitizedId);
          }
        }
      }
    });
    
    return new GraphQLSchema({
      query: QueryType
    });
  }
  
  async getUser(id) {
    return {
      id,
      email: 'user@enbd.com',
      name: 'User',
      ssn: '123-45-6789'
    };
  }
}

/**
 * GraphQL Rate Limiting
 */
class GraphQLRateLimiter {
  constructor() {
    this.redis = new Redis();
  }
  
  async checkRateLimit(userId, cost) {
    const key = `graphql:ratelimit:${userId}`;
    const maxPoints = 1000; // Max complexity points per hour
    
    const current = await this.redis.incrby(key, cost);
    
    if (current === cost) {
      await this.redis.expire(key, 3600);
    }
    
    if (current > maxPoints) {
      throw new Error('GraphQL rate limit exceeded');
    }
    
    return {
      remaining: maxPoints - current
    };
  }
}

/**
 * ENBD GraphQL API
 */
class ENBDGraphQLAPI {
  constructor() {
    this.securityService = new GraphQLSecurityService();
    this.schemaBuilder = new SecureGraphQLSchema();
    this.rateLimiter = new GraphQLRateLimiter();
  }
  
  setup(app) {
    app.use('/graphql',
      this.authenticate,
      graphqlHTTP(async (req) => ({
        schema: this.schemaBuilder.createSchema(),
        context: {
          user: req.user
        },
        graphiql: process.env.NODE_ENV !== 'production',
        validationRules: [
          this.securityService.getDepthLimitRule(),
          this.securityService.getComplexityLimitRule()
        ],
        ...this.securityService.disableIntrospection()
      }))
    );
  }
  
  authenticate(req, res, next) {
    req.user = { id: 'user-123', role: 'customer' };
    next();
  }
}

module.exports = {
  GraphQLSecurityService,
  SecureGraphQLSchema,
  GraphQLRateLimiter,
  ENBDGraphQLAPI
};
```

**GraphQL Security Checklist:**

1. ✅ **Query depth limiting** (prevent nested attacks)
2. ✅ **Query complexity limiting**
3. ✅ **Disable introspection in production**
4. ✅ **Field-level authorization**
5. ✅ **Input sanitization**
6. ✅ **Rate limiting by complexity**
7. ✅ **Query timeouts**

---

### Q38. How do you implement secure WebSocket connections?

**Answer:**

```javascript
const WebSocket = require('ws');
const jwt = require('jsonwebtoken');

/**
 * Secure WebSocket Service
 */
class SecureWebSocketService {
  constructor() {
    this.wss = null;
    this.connections = new Map();
  }
  
  /**
   * Create secure WebSocket server
   */
  createServer(server) {
    this.wss = new WebSocket.Server({
      server,
      
      // Verify client on connection
      verifyClient: async (info, callback) => {
        try {
          await this.authenticateWebSocket(info);
          callback(true);
        } catch (error) {
          callback(false, 401, 'Unauthorized');
        }
      },
      
      // Connection limit
      maxPayload: 100 * 1024, // 100KB max message size
      
      // Timeout settings
      clientTracking: true
    });
    
    this.setupConnectionHandlers();
  }
  
  /**
   * Authenticate WebSocket connection
   */
  async authenticateWebSocket(info) {
    const url = new URL(info.req.url, 'wss://localhost');
    const token = url.searchParams.get('token');
    
    if (!token) {
      throw new Error('No authentication token');
    }
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      info.req.user = decoded;
      return decoded;
    } catch (error) {
      throw new Error('Invalid token');
    }
  }
  
  /**
   * Setup connection handlers
   */
  setupConnectionHandlers() {
    this.wss.on('connection', (ws, req) => {
      const userId = req.user.id;
      
      // Store connection
      this.connections.set(userId, ws);
      
      // Setup message handler
      ws.on('message', async (message) => {
        try {
          await this.handleMessage(ws, userId, message);
        } catch (error) {
          ws.send(JSON.stringify({
            type: 'error',
            message: error.message
          }));
        }
      });
      
      // Setup ping/pong (heartbeat)
      ws.isAlive = true;
      ws.on('pong', () => {
        ws.isAlive = true;
      });
      
      // Handle disconnect
      ws.on('close', () => {
        this.connections.delete(userId);
      });
      
      // Send welcome message
      ws.send(JSON.stringify({
        type: 'connected',
        message: 'WebSocket connection established'
      }));
    });
    
    // Heartbeat interval
    this.setupHeartbeat();
  }
  
  /**
   * Handle incoming messages
   */
  async handleMessage(ws, userId, message) {
    // Rate limiting
    if (!await this.checkMessageRateLimit(userId)) {
      throw new Error('Rate limit exceeded');
    }
    
    // Parse message
    let data;
    try {
      data = JSON.parse(message);
    } catch (error) {
      throw new Error('Invalid message format');
    }
    
    // Validate message structure
    if (!data.type || !data.payload) {
      throw new Error('Invalid message structure');
    }
    
    // Sanitize input
    data.payload = this.sanitizePayload(data.payload);
    
    // Route message
    switch (data.type) {
      case 'subscribe':
        await this.handleSubscribe(ws, userId, data.payload);
        break;
      
      case 'message':
        await this.handleChatMessage(ws, userId, data.payload);
        break;
      
      default:
        throw new Error('Unknown message type');
    }
  }
  
  /**
   * Message rate limiting
   */
  async checkMessageRateLimit(userId) {
    const key = `ws:ratelimit:${userId}`;
    const count = await this.redis.incr(key);
    
    if (count === 1) {
      await this.redis.expire(key, 60);
    }
    
    return count <= 100; // 100 messages per minute
  }
  
  /**
   * Sanitize message payload
   */
  sanitizePayload(payload) {
    if (typeof payload === 'string') {
      return payload.trim().substring(0, 1000);
    }
    
    if (typeof payload === 'object') {
      const sanitized = {};
      for (const [key, value] of Object.entries(payload)) {
        sanitized[key] = this.sanitizePayload(value);
      }
      return sanitized;
    }
    
    return payload;
  }
  
  /**
   * Heartbeat to detect dead connections
   */
  setupHeartbeat() {
    setInterval(() => {
      this.wss.clients.forEach((ws) => {
        if (ws.isAlive === false) {
          return ws.terminate();
        }
        
        ws.isAlive = false;
        ws.ping();
      });
    }, 30000); // 30 seconds
  }
  
  /**
   * Broadcast to specific users
   */
  broadcast(userIds, message) {
    userIds.forEach(userId => {
      const ws = this.connections.get(userId);
      if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify(message));
      }
    });
  }
  
  async handleSubscribe(ws, userId, payload) {
    // Subscribe to channel
  }
  
  async handleChatMessage(ws, userId, payload) {
    // Handle chat message
  }
}

/**
 * ENBD Real-time Notifications via WebSocket
 */
class ENBDRealtimeNotifications {
  constructor() {
    this.wsService = new SecureWebSocketService();
  }
  
  setup(server) {
    this.wsService.createServer(server);
  }
  
  /**
   * Send transaction notification
   */
  async notifyTransaction(userId, transaction) {
    const ws = this.wsService.connections.get(userId);
    
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: 'transaction',
        data: {
          id: transaction.id,
          amount: transaction.amount,
          type: transaction.type,
          timestamp: transaction.timestamp
        }
      }));
    }
  }
  
  /**
   * Send security alert
   */
  async sendSecurityAlert(userId, alert) {
    const ws = this.wsService.connections.get(userId);
    
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: 'security_alert',
        severity: 'high',
        data: alert
      }));
    }
  }
}

module.exports = {
  SecureWebSocketService,
  ENBDRealtimeNotifications
};
```

**WebSocket Security Best Practices:**

1. ✅ **Authentication on connection**
2. ✅ **Message rate limiting**
3. ✅ **Input validation and sanitization**
4. ✅ **Message size limits**
5. ✅ **Heartbeat/ping-pong**
6. ✅ **Connection limits per user**
7. ✅ **Secure WebSocket (wss://) only**

---

### Q39. How do you implement Content Security Policy (CSP)?

**Answer:**

```javascript
/**
 * Content Security Policy Service
 */
class ContentSecurityPolicyService {
  /**
   * Get strict CSP configuration
   */
  static getStrictCSP() {
    return {
      directives: {
        // Default fallback
        defaultSrc: ["'self'"],
        
        // Scripts
        scriptSrc: [
          "'self'",
          "'nonce-{NONCE}'", // Dynamic nonce
          "https://cdn.enbd.com"
        ],
        
        // No eval() or inline scripts
        'script-src-attr': ["'none'"],
        
        // Styles
        styleSrc: [
          "'self'",
          "'nonce-{NONCE}'",
          "https://cdn.enbd.com",
          "https://fonts.googleapis.com"
        ],
        
        // Images
        imgSrc: [
          "'self'",
          "data:", // For base64 images
          "https:",
          "blob:" // For generated images
        ],
        
        // Fonts
        fontSrc: [
          "'self'",
          "https://fonts.gstatic.com"
        ],
        
        // AJAX, WebSocket
        connectSrc: [
          "'self'",
          "https://api.enbd.com",
          "wss://ws.enbd.com"
        ],
        
        // Media
        mediaSrc: ["'self'"],
        
        // Objects
        objectSrc: ["'none'"],
        
        // Frames
        frameSrc: ["'none'"],
        
        // Frame ancestors (prevents clickjacking)
        frameAncestors: ["'none'"],
        
        // Base URI
        baseUri: ["'self'"],
        
        // Form actions
        formAction: ["'self'"],
        
        // Upgrade insecure requests
        upgradeInsecureRequests: [],
        
        // Block mixed content
        blockAllMixedContent: [],
        
        // Require SRI for scripts and styles
        requireSriFor: ['script', 'style']
      }
    };
  }
  
  /**
   * Generate nonce for inline scripts
   */
  static generateNonce() {
    return crypto.randomBytes(16).toString('base64');
  }
  
  /**
   * CSP middleware
   */
  static cspMiddleware() {
    return (req, res, next) => {
      const nonce = ContentSecurityPolicyService.generateNonce();
      res.locals.nonce = nonce;
      
      const csp = ContentSecurityPolicyService.getStrictCSP();
      
      // Replace {NONCE} with actual nonce
      let cspString = '';
      for (const [directive, values] of Object.entries(csp.directives)) {
        const directiveName = directive.replace(/([A-Z])/g, '-$1').toLowerCase();
        const valuesString = values.map(v => v.replace('{NONCE}', nonce)).join(' ');
        cspString += `${directiveName} ${valuesString}; `;
      }
      
      res.setHeader('Content-Security-Policy', cspString.trim());
      
      next();
    };
  }
  
  /**
   * CSP violation reporting
   */
  static setupViolationReporting(app) {
    app.post('/api/csp-violation-report', express.json({ type: 'application/csp-report' }), (req, res) => {
      const report = req.body['csp-report'];
      
      console.error('CSP Violation:', {
        documentUri: report['document-uri'],
        violatedDirective: report['violated-directive'],
        blockedUri: report['blocked-uri'],
        sourceFile: report['source-file'],
        lineNumber: report['line-number']
      });
      
      // Store violation for analysis
      // Alert security team if needed
      
      res.status(204).send();
    });
  }
  
  /**
   * Add report-uri to CSP
   */
  static addReporting() {
    return {
      ...this.getStrictCSP(),
      directives: {
        ...this.getStrictCSP().directives,
        reportUri: ['/api/csp-violation-report']
      }
    };
  }
}

/**
 * Subresource Integrity (SRI) Helper
 */
class SubresourceIntegrityService {
  /**
   * Generate SRI hash for file
   */
  static generateSRIHash(fileContent) {
    const hash = crypto.createHash('sha384').update(fileContent).digest('base64');
    return `sha384-${hash}`;
  }
  
  /**
   * SRI script tag
   */
  static scriptTag(src, integrity) {
    return `<script src="${src}" integrity="${integrity}" crossorigin="anonymous"></script>`;
  }
  
  /**
   * SRI stylesheet tag
   */
  static styleTag(href, integrity) {
    return `<link rel="stylesheet" href="${href}" integrity="${integrity}" crossorigin="anonymous">`;
  }
}

module.exports = {
  ContentSecurityPolicyService,
  SubresourceIntegrityService
};
```

**CSP Best Practices:**

1. ✅ **Start with strict policy**
2. ✅ **Use nonces for inline scripts**
3. ✅ **No 'unsafe-inline' or 'unsafe-eval'**
4. ✅ **Whitelist specific domains**
5. ✅ **Report violations**
6. ✅ **Use SRI for external resources**
7. ✅ **Frame-ancestors to prevent clickjacking**

---

### Q40. How do you implement security testing in CI/CD?

**Answer:**

```javascript
/**
 * Security Testing CI/CD Integration
 */
class SecurityTestingService {
  /**
   * SAST (Static Application Security Testing)
   */
  static getSASTConfig() {
    return {
      tools: [
        {
          name: 'ESLint Security Plugin',
          command: 'eslint . --ext .js --plugin security',
          config: {
            plugins: ['security'],
            rules: {
              'security/detect-object-injection': 'error',
              'security/detect-non-literal-regexp': 'error',
              'security/detect-unsafe-regex': 'error',
              'security/detect-buffer-noassert': 'error',
              'security/detect-child-process': 'error',
              'security/detect-disable-mustache-escape': 'error',
              'security/detect-eval-with-expression': 'error',
              'security/detect-no-csrf-before-method-override': 'error',
              'security/detect-non-literal-fs-filename': 'error',
              'security/detect-non-literal-require': 'error',
              'security/detect-possible-timing-attacks': 'error',
              'security/detect-pseudoRandomBytes': 'error'
            }
          }
        },
        {
          name: 'SonarQube',
          command: 'sonar-scanner',
          config: {
            'sonar.projectKey': 'enbd-banking',
            'sonar.sources': 'src',
            'sonar.host.url': 'https://sonar.enbd.com'
          }
        },
        {
          name: 'Snyk Code',
          command: 'snyk code test',
          failOn: 'high'
        }
      ]
    };
  }
  
  /**
   * Dependency Scanning (SCA)
   */
  static getDependencyScanConfig() {
    return {
      tools: [
        {
          name: 'npm audit',
          command: 'npm audit --audit-level=high',
          fix: 'npm audit fix'
        },
        {
          name: 'Snyk',
          command: 'snyk test --severity-threshold=high',
          monitor: 'snyk monitor'
        },
        {
          name: 'OWASP Dependency Check',
          command: 'dependency-check --project enbd --scan . --format JSON'
        }
      ]
    };
  }
  
  /**
   * DAST (Dynamic Application Security Testing)
   */
  static getDASTConfig() {
    return {
      tools: [
        {
          name: 'OWASP ZAP',
          command: 'zap-baseline.py -t https://staging.enbd.com',
          config: {
            rules: ['10020', '10021', '10023'] // XSS, SQLi, etc.
          }
        },
        {
          name: 'Burp Suite',
          api: 'https://burp.enbd.com/api/scan',
          target: 'https://staging.enbd.com'
        }
      ]
    };
  }
  
  /**
   * Container Security Scanning
   */
  static getContainerScanConfig() {
    return {
      tools: [
        {
          name: 'Trivy',
          command: 'trivy image enbd/banking:latest',
          severity: ['CRITICAL', 'HIGH']
        },
        {
          name: 'Clair',
          command: 'clairctl analyze enbd/banking:latest'
        },
        {
          name: 'Anchore',
          command: 'anchore-cli image scan enbd/banking:latest'
        }
      ]
    };
  }
  
  /**
   * Secrets Scanning
   */
  static getSecretscanConfig() {
    return {
      tools: [
        {
          name: 'GitGuardian',
          command: 'ggshield scan path .'
        },
        {
          name: 'TruffleHog',
          command: 'trufflehog filesystem . --json'
        },
        {
          name: 'git-secrets',
          command: 'git secrets --scan'
        }
      ]
    };
  }
}

/**
 * GitHub Actions Security Workflow
 */
const githubActionsWorkflow = `
name: Security Testing

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      # SAST - Static Analysis
      - name: Run ESLint Security
        run: |
          npm install
          npm run lint:security
      
      # Dependency Scanning
      - name: Run npm audit
        run: npm audit --audit-level=high
      
      - name: Run Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: \${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      # Secrets Scanning
      - name: Run TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: main
          head: HEAD
      
      # Container Scanning
      - name: Build Docker Image
        run: docker build -t enbd/banking:${{ github.sha }} .
      
      - name: Run Trivy Scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: enbd/banking:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      # DAST - Dynamic Testing
      - name: Run OWASP ZAP Scan
        uses: zaproxy/action-baseline@v0.7.0
        with:
          target: 'https://staging.enbd.com'
          rules_file_name: '.zap/rules.tsv'
      
      # Security Report
      - name: Upload Security Report
        uses: actions/upload-artifact@v3
        with:
          name: security-report
          path: security-report.json
`;

/**
 * Jenkins Security Pipeline
 */
const jenkinsPipeline = `
pipeline {
  agent any
  
  stages {
    stage('Security Checks') {
      parallel {
        stage('SAST') {
          steps {
            sh 'npm run lint:security'
            sh 'sonar-scanner'
          }
        }
        
        stage('Dependency Scan') {
          steps {
            sh 'npm audit --audit-level=high'
            sh 'snyk test --severity-threshold=high'
          }
        }
        
        stage('Secrets Scan') {
          steps {
            sh 'trufflehog filesystem . --json > secrets-report.json'
          }
        }
      }
    }
    
    stage('Build & Container Scan') {
      steps {
        sh 'docker build -t enbd/banking:latest .'
        sh 'trivy image enbd/banking:latest'
      }
    }
    
    stage('DAST') {
      when {
        branch 'staging'
      }
      steps {
        sh 'zap-baseline.py -t https://staging.enbd.com'
      }
    }
  }
  
  post {
    always {
      publishHTML([
        reportName: 'Security Report',
        reportDir: 'reports',
        reportFiles: 'security.html'
      ])
    }
    
    failure {
      mail to: 'security@enbd.com',
           subject: "Security scan failed: ${env.JOB_NAME}",
           body: "Check console output at ${env.BUILD_URL}"
    }
  }
}
`;

module.exports = {
  SecurityTestingService,
  githubActionsWorkflow,
  jenkinsPipeline
};
```

**Security Testing in CI/CD:**

1. ✅ **SAST** (ESLint Security, SonarQube, Snyk Code)
2. ✅ **Dependency Scanning** (npm audit, Snyk, OWASP Dependency Check)
3. ✅ **Secrets Scanning** (TruffleHog, GitGuardian, git-secrets)
4. ✅ **Container Scanning** (Trivy, Clair, Anchore)
5. ✅ **DAST** (OWASP ZAP, Burp Suite)
6. ✅ **Automated security gates** (fail build on critical)
7. ✅ **Security reports** and notifications

---

**Summary Q36-Q40:**
- API versioning & deprecation (headers, sunset dates) ✅
- Secure GraphQL (depth/complexity limits, field auth) ✅
- Secure WebSockets (authentication, rate limiting) ✅
- Content Security Policy (strict CSP, nonces, SRI) ✅
- Security testing in CI/CD (SAST, DAST, SCA) ✅

Continuing with final security questions...
