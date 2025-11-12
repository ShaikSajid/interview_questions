# APIs Questions 12-14: Security & Monitoring

---

### Q12. How do you implement OAuth 2.0 flows for API security?

**Answer:**

```javascript
/**
 * OAuth 2.0 Implementation for Banking API
 */
const crypto = require('crypto');
const jwt = require('jsonwebtoken');
const { promisify } = require('util');
const randomBytes = promisify(crypto.randomBytes);

/**
 * OAuth 2.0 Authorization Server
 */
class OAuth2Server {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
    
    // Client credentials
    this.clients = new Map();
    
    // Authorization codes
    this.authCodes = new Map();
    
    // Access tokens
    this.accessTokens = new Map();
    
    // Refresh tokens
    this.refreshTokens = new Map();
  }
  
  /**
   * Register OAuth client
   */
  async registerClient(name, redirectUris, grantTypes = ['authorization_code']) {
    const clientId = `enbd_${crypto.randomBytes(16).toString('hex')}`;
    const clientSecret = crypto.randomBytes(32).toString('hex');
    
    const client = {
      clientId,
      clientSecret: await this.hashSecret(clientSecret),
      name,
      redirectUris,
      grantTypes,
      createdAt: new Date()
    };
    
    await this.db.query(`
      INSERT INTO oauth_clients (client_id, client_secret, name, redirect_uris, grant_types)
      VALUES ($1, $2, $3, $4, $5)
    `, [clientId, client.clientSecret, name, JSON.stringify(redirectUris), JSON.stringify(grantTypes)]);
    
    return { clientId, clientSecret }; // Return plain secret only once
  }
  
  /**
   * 1. Authorization Code Flow (most secure for web apps)
   */
  async authorize(req, res) {
    const {
      client_id,
      redirect_uri,
      response_type,
      scope,
      state
    } = req.query;
    
    // Validate client
    const client = await this.validateClient(client_id, redirect_uri);
    if (!client) {
      return res.status(400).json({
        error: 'invalid_client',
        error_description: 'Invalid client_id or redirect_uri'
      });
    }
    
    // Validate response_type
    if (response_type !== 'code') {
      return res.status(400).json({
        error: 'unsupported_response_type',
        error_description: 'Only authorization code flow is supported'
      });
    }
    
    // Render authorization page
    res.render('authorize', {
      client,
      scope,
      state,
      client_id,
      redirect_uri
    });
  }
  
  /**
   * User grants authorization
   */
  async grant(req, res) {
    const {
      client_id,
      redirect_uri,
      scope,
      state
    } = req.body;
    
    const userId = req.user.id; // From session
    
    // Generate authorization code
    const code = crypto.randomBytes(32).toString('hex');
    
    const authCode = {
      code,
      clientId: client_id,
      userId,
      redirectUri: redirect_uri,
      scope,
      expiresAt: Date.now() + 600000, // 10 minutes
      used: false
    };
    
    // Store authorization code
    await this.redis.setex(
      `auth_code:${code}`,
      600, // 10 minutes TTL
      JSON.stringify(authCode)
    );
    
    // Redirect back with code
    const redirectUrl = new URL(redirect_uri);
    redirectUrl.searchParams.set('code', code);
    if (state) redirectUrl.searchParams.set('state', state);
    
    res.redirect(redirectUrl.toString());
  }
  
  /**
   * Exchange authorization code for access token
   */
  async token(req, res) {
    const {
      grant_type,
      code,
      redirect_uri,
      client_id,
      client_secret,
      refresh_token
    } = req.body;
    
    // Validate client credentials
    const client = await this.authenticateClient(client_id, client_secret);
    if (!client) {
      return res.status(401).json({
        error: 'invalid_client',
        error_description: 'Client authentication failed'
      });
    }
    
    if (grant_type === 'authorization_code') {
      return this.handleAuthorizationCodeGrant(res, code, redirect_uri, client);
    } else if (grant_type === 'refresh_token') {
      return this.handleRefreshTokenGrant(res, refresh_token, client);
    } else if (grant_type === 'client_credentials') {
      return this.handleClientCredentialsGrant(res, client);
    } else {
      return res.status(400).json({
        error: 'unsupported_grant_type'
      });
    }
  }
  
  /**
   * Handle authorization code grant
   */
  async handleAuthorizationCodeGrant(res, code, redirectUri, client) {
    // Retrieve authorization code
    const authCodeData = await this.redis.get(`auth_code:${code}`);
    
    if (!authCodeData) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Authorization code is invalid or expired'
      });
    }
    
    const authCode = JSON.parse(authCodeData);
    
    // Validate
    if (authCode.used) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Authorization code has already been used'
      });
    }
    
    if (authCode.clientId !== client.clientId) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Authorization code was issued to different client'
      });
    }
    
    if (authCode.redirectUri !== redirectUri) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Redirect URI mismatch'
      });
    }
    
    // Mark as used
    authCode.used = true;
    await this.redis.setex(`auth_code:${code}`, 60, JSON.stringify(authCode));
    
    // Generate tokens
    const tokens = await this.generateTokens(authCode.userId, client.clientId, authCode.scope);
    
    res.json({
      access_token: tokens.accessToken,
      token_type: 'Bearer',
      expires_in: 3600,
      refresh_token: tokens.refreshToken,
      scope: authCode.scope
    });
  }
  
  /**
   * 2. Client Credentials Flow (for service-to-service)
   */
  async handleClientCredentialsGrant(res, client) {
    // Generate access token (no user context)
    const accessToken = jwt.sign(
      {
        client_id: client.clientId,
        grant_type: 'client_credentials',
        scope: 'api:read api:write'
      },
      process.env.JWT_SECRET,
      {
        expiresIn: '1h',
        issuer: 'enbd-oauth',
        audience: 'enbd-api'
      }
    );
    
    res.json({
      access_token: accessToken,
      token_type: 'Bearer',
      expires_in: 3600,
      scope: 'api:read api:write'
    });
  }
  
  /**
   * 3. Refresh Token Flow
   */
  async handleRefreshTokenGrant(res, refreshToken, client) {
    // Verify refresh token
    let decoded;
    try {
      decoded = jwt.verify(refreshToken, process.env.REFRESH_TOKEN_SECRET);
    } catch (error) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Invalid refresh token'
      });
    }
    
    // Check if refresh token is revoked
    const revoked = await this.redis.get(`revoked_token:${refreshToken}`);
    if (revoked) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Refresh token has been revoked'
      });
    }
    
    // Generate new access token
    const accessToken = jwt.sign(
      {
        userId: decoded.userId,
        customerId: decoded.customerId,
        scope: decoded.scope
      },
      process.env.JWT_SECRET,
      {
        expiresIn: '1h',
        issuer: 'enbd-oauth',
        audience: 'enbd-api'
      }
    );
    
    res.json({
      access_token: accessToken,
      token_type: 'Bearer',
      expires_in: 3600
    });
  }
  
  /**
   * 4. PKCE (Proof Key for Code Exchange) for mobile/SPA
   */
  async authorizeWithPKCE(req, res) {
    const {
      client_id,
      redirect_uri,
      code_challenge,
      code_challenge_method,
      state
    } = req.query;
    
    // Validate code_challenge_method
    if (code_challenge_method !== 'S256') {
      return res.status(400).json({
        error: 'invalid_request',
        error_description: 'Only S256 code challenge method is supported'
      });
    }
    
    // Store code challenge
    const code = crypto.randomBytes(32).toString('hex');
    
    await this.redis.setex(
      `pkce:${code}`,
      600,
      JSON.stringify({
        codeChallenge: code_challenge,
        codeChallengeMethod: code_challenge_method,
        clientId: client_id,
        redirectUri: redirect_uri,
        userId: req.user.id
      })
    );
    
    // Redirect with code
    const redirectUrl = new URL(redirect_uri);
    redirectUrl.searchParams.set('code', code);
    if (state) redirectUrl.searchParams.set('state', state);
    
    res.redirect(redirectUrl.toString());
  }
  
  /**
   * Token endpoint with PKCE verification
   */
  async tokenWithPKCE(req, res) {
    const { code, code_verifier, client_id } = req.body;
    
    // Retrieve PKCE data
    const pkceData = await this.redis.get(`pkce:${code}`);
    if (!pkceData) {
      return res.status(400).json({
        error: 'invalid_grant'
      });
    }
    
    const pkce = JSON.parse(pkceData);
    
    // Verify code_verifier
    const hash = crypto.createHash('sha256').update(code_verifier).digest('base64url');
    
    if (hash !== pkce.codeChallenge) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Code verifier does not match challenge'
      });
    }
    
    // Generate tokens
    const tokens = await this.generateTokens(pkce.userId, client_id, 'full_access');
    
    res.json({
      access_token: tokens.accessToken,
      token_type: 'Bearer',
      expires_in: 3600,
      refresh_token: tokens.refreshToken
    });
  }
  
  /**
   * Token introspection endpoint
   */
  async introspect(req, res) {
    const { token } = req.body;
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      
      res.json({
        active: true,
        scope: decoded.scope,
        client_id: decoded.client_id,
        username: decoded.username,
        exp: decoded.exp,
        iat: decoded.iat
      });
    } catch (error) {
      res.json({ active: false });
    }
  }
  
  /**
   * Token revocation endpoint
   */
  async revoke(req, res) {
    const { token, token_type_hint } = req.body;
    
    // Add to revocation list
    if (token_type_hint === 'refresh_token') {
      // Revoke for 7 days (refresh token lifetime)
      await this.redis.setex(`revoked_token:${token}`, 604800, '1');
    } else {
      // Revoke for 1 hour (access token lifetime)
      await this.redis.setex(`revoked_token:${token}`, 3600, '1');
    }
    
    res.status(200).json({ success: true });
  }
  
  /**
   * Helper: Generate access and refresh tokens
   */
  async generateTokens(userId, clientId, scope) {
    // Get user info
    const user = await this.db.query('SELECT * FROM customers WHERE id = $1', [userId]);
    const userData = user.rows[0];
    
    const accessToken = jwt.sign(
      {
        userId,
        customerId: userData.id,
        email: userData.email,
        scope,
        client_id: clientId
      },
      process.env.JWT_SECRET,
      {
        expiresIn: '1h',
        issuer: 'enbd-oauth',
        audience: 'enbd-api'
      }
    );
    
    const refreshToken = jwt.sign(
      {
        userId,
        customerId: userData.id,
        scope,
        client_id: clientId,
        type: 'refresh'
      },
      process.env.REFRESH_TOKEN_SECRET,
      {
        expiresIn: '7d',
        issuer: 'enbd-oauth'
      }
    );
    
    return { accessToken, refreshToken };
  }
  
  /**
   * Helper: Validate client
   */
  async validateClient(clientId, redirectUri) {
    const result = await this.db.query(
      'SELECT * FROM oauth_clients WHERE client_id = $1',
      [clientId]
    );
    
    if (result.rows.length === 0) return null;
    
    const client = result.rows[0];
    const redirectUris = JSON.parse(client.redirect_uris);
    
    if (!redirectUris.includes(redirectUri)) return null;
    
    return client;
  }
  
  /**
   * Helper: Authenticate client
   */
  async authenticateClient(clientId, clientSecret) {
    const result = await this.db.query(
      'SELECT * FROM oauth_clients WHERE client_id = $1',
      [clientId]
    );
    
    if (result.rows.length === 0) return null;
    
    const client = result.rows[0];
    
    // Verify secret
    const valid = await this.verifySecret(clientSecret, client.client_secret);
    if (!valid) return null;
    
    return client;
  }
  
  async hashSecret(secret) {
    return crypto.createHash('sha256').update(secret).digest('hex');
  }
  
  async verifySecret(plain, hashed) {
    return this.hashSecret(plain) === hashed;
  }
}

/**
 * OAuth 2.0 Middleware
 */
function requireOAuth(scopes = []) {
  return async (req, res, next) => {
    const authHeader = req.headers['authorization'];
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        error: 'invalid_token',
        error_description: 'Missing or invalid authorization header'
      });
    }
    
    const token = authHeader.substring(7);
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET, {
        issuer: 'enbd-oauth',
        audience: 'enbd-api'
      });
      
      // Check if token is revoked
      const revoked = await redis.get(`revoked_token:${token}`);
      if (revoked) {
        return res.status(401).json({
          error: 'invalid_token',
          error_description: 'Token has been revoked'
        });
      }
      
      // Check scopes
      if (scopes.length > 0) {
        const tokenScopes = decoded.scope ? decoded.scope.split(' ') : [];
        const hasRequiredScope = scopes.some(scope => tokenScopes.includes(scope));
        
        if (!hasRequiredScope) {
          return res.status(403).json({
            error: 'insufficient_scope',
            error_description: `Required scopes: ${scopes.join(', ')}`
          });
        }
      }
      
      req.user = decoded;
      next();
      
    } catch (error) {
      return res.status(401).json({
        error: 'invalid_token',
        error_description: error.message
      });
    }
  };
}

/**
 * Express Routes
 */
const express = require('express');
const router = express.Router();

// Authorization endpoint
router.get('/oauth/authorize', sessionMiddleware, oauth2Server.authorize.bind(oauth2Server));
router.post('/oauth/grant', sessionMiddleware, oauth2Server.grant.bind(oauth2Server));

// Token endpoint
router.post('/oauth/token', oauth2Server.token.bind(oauth2Server));

// PKCE endpoints
router.get('/oauth/authorize-pkce', sessionMiddleware, oauth2Server.authorizeWithPKCE.bind(oauth2Server));
router.post('/oauth/token-pkce', oauth2Server.tokenWithPKCE.bind(oauth2Server));

// Token management
router.post('/oauth/introspect', oauth2Server.introspect.bind(oauth2Server));
router.post('/oauth/revoke', oauth2Server.revoke.bind(oauth2Server));

// Protected API routes
router.get('/api/accounts', 
  requireOAuth(['accounts:read']), 
  async (req, res) => {
    // Access user info from req.user
    res.json({ accounts: [] });
  }
);

module.exports = { OAuth2Server, requireOAuth };
```

---

### Q13. How do you implement API monitoring and observability?

**Answer:**

```javascript
/**
 * API Monitoring and Observability
 */
const prometheus = require('prom-client');
const winston = require('winston');
const { v4: uuidv4 } = require('uuid');

/**
 * Prometheus Metrics
 */
class MetricsService {
  constructor() {
    // Create a Registry
    this.register = new prometheus.Registry();
    
    // Add default metrics
    prometheus.collectDefaultMetrics({ register: this.register });
    
    // Custom metrics
    this.httpRequestDuration = new prometheus.Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status_code'],
      buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5, 10]
    });
    
    this.httpRequestTotal = new prometheus.Counter({
      name: 'http_requests_total',
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'route', 'status_code']
    });
    
    this.httpRequestsInProgress = new prometheus.Gauge({
      name: 'http_requests_in_progress',
      help: 'Number of HTTP requests currently in progress',
      labelNames: ['method', 'route']
    });
    
    // Business metrics
    this.transactionAmount = new prometheus.Histogram({
      name: 'transaction_amount_aed',
      help: 'Transaction amount in AED',
      labelNames: ['type', 'status'],
      buckets: [100, 500, 1000, 5000, 10000, 50000, 100000]
    });
    
    this.transactionTotal = new prometheus.Counter({
      name: 'transactions_total',
      help: 'Total number of transactions',
      labelNames: ['type', 'status']
    });
    
    this.accountBalance = new prometheus.Gauge({
      name: 'account_balance_aed',
      help: 'Current account balance in AED',
      labelNames: ['account_type']
    });
    
    this.activeUsers = new prometheus.Gauge({
      name: 'active_users',
      help: 'Number of active users'
    });
    
    // Error tracking
    this.errorTotal = new prometheus.Counter({
      name: 'errors_total',
      help: 'Total number of errors',
      labelNames: ['error_type', 'route']
    });
    
    // Register metrics
    this.register.registerMetric(this.httpRequestDuration);
    this.register.registerMetric(this.httpRequestTotal);
    this.register.registerMetric(this.httpRequestsInProgress);
    this.register.registerMetric(this.transactionAmount);
    this.register.registerMetric(this.transactionTotal);
    this.register.registerMetric(this.accountBalance);
    this.register.registerMetric(this.activeUsers);
    this.register.registerMetric(this.errorTotal);
  }
  
  /**
   * Get metrics in Prometheus format
   */
  async getMetrics() {
    return this.register.metrics();
  }
}

/**
 * Request Tracking Middleware
 */
function trackingMiddleware(metricsService) {
  return (req, res, next) => {
    // Generate request ID
    req.id = req.get('X-Request-ID') || uuidv4();
    res.set('X-Request-ID', req.id);
    
    // Start timer
    const start = Date.now();
    
    // Track in-progress requests
    const route = req.route?.path || req.path;
    metricsService.httpRequestsInProgress.inc({ 
      method: req.method, 
      route 
    });
    
    // Capture response
    res.on('finish', () => {
      const duration = (Date.now() - start) / 1000;
      
      // Record metrics
      metricsService.httpRequestDuration.observe(
        { 
          method: req.method, 
          route, 
          status_code: res.statusCode 
        },
        duration
      );
      
      metricsService.httpRequestTotal.inc({
        method: req.method,
        route,
        status_code: res.statusCode
      });
      
      metricsService.httpRequestsInProgress.dec({
        method: req.method,
        route
      });
      
      // Log request
      logger.info('HTTP Request', {
        requestId: req.id,
        method: req.method,
        url: req.url,
        route,
        statusCode: res.statusCode,
        duration: `${duration.toFixed(3)}s`,
        userAgent: req.get('user-agent'),
        ip: req.ip,
        userId: req.user?.userId
      });
    });
    
    next();
  };
}

/**
 * Distributed Tracing with Correlation IDs
 */
class TracingService {
  constructor() {
    this.spans = new Map();
  }
  
  /**
   * Start a new span
   */
  startSpan(name, parentSpanId = null) {
    const spanId = uuidv4();
    const span = {
      spanId,
      parentSpanId,
      name,
      startTime: Date.now(),
      tags: {},
      logs: []
    };
    
    this.spans.set(spanId, span);
    return spanId;
  }
  
  /**
   * Add tag to span
   */
  setTag(spanId, key, value) {
    const span = this.spans.get(spanId);
    if (span) {
      span.tags[key] = value;
    }
  }
  
  /**
   * Add log to span
   */
  log(spanId, message, data = {}) {
    const span = this.spans.get(spanId);
    if (span) {
      span.logs.push({
        timestamp: Date.now(),
        message,
        data
      });
    }
  }
  
  /**
   * Finish span
   */
  finishSpan(spanId) {
    const span = this.spans.get(spanId);
    if (span) {
      span.endTime = Date.now();
      span.duration = span.endTime - span.startTime;
      
      // Send to tracing backend (Jaeger, Zipkin, etc.)
      this.sendTrace(span);
      
      this.spans.delete(spanId);
    }
  }
  
  /**
   * Send trace to backend
   */
  sendTrace(span) {
    // Send to Jaeger/Zipkin
    console.log('Trace:', JSON.stringify(span, null, 2));
  }
}

/**
 * APM (Application Performance Monitoring)
 */
class APMService {
  constructor() {
    this.transactions = [];
    this.errors = [];
    this.slowQueries = [];
  }
  
  /**
   * Track transaction
   */
  trackTransaction(name, duration, metadata = {}) {
    this.transactions.push({
      name,
      duration,
      timestamp: new Date(),
      metadata
    });
    
    // Alert on slow transactions
    if (duration > 1000) { // > 1 second
      this.alertSlowTransaction(name, duration, metadata);
    }
  }
  
  /**
   * Track error
   */
  trackError(error, context = {}) {
    this.errors.push({
      message: error.message,
      stack: error.stack,
      name: error.name,
      context,
      timestamp: new Date()
    });
    
    // Send to error tracking service (Sentry, etc.)
    this.sendToSentry(error, context);
  }
  
  /**
   * Track database query
   */
  trackQuery(query, duration, params = []) {
    if (duration > 500) { // > 500ms
      this.slowQueries.push({
        query,
        duration,
        params,
        timestamp: new Date()
      });
      
      logger.warn('Slow query detected', {
        query,
        duration: `${duration}ms`,
        params
      });
    }
  }
  
  /**
   * Get APM summary
   */
  getSummary() {
    const last5Minutes = Date.now() - 300000;
    
    const recentTransactions = this.transactions.filter(
      t => t.timestamp.getTime() > last5Minutes
    );
    
    const avgDuration = recentTransactions.length > 0
      ? recentTransactions.reduce((sum, t) => sum + t.duration, 0) / recentTransactions.length
      : 0;
    
    return {
      transactions: {
        total: recentTransactions.length,
        avgDuration: `${avgDuration.toFixed(2)}ms`,
        slowCount: recentTransactions.filter(t => t.duration > 1000).length
      },
      errors: {
        total: this.errors.filter(e => e.timestamp.getTime() > last5Minutes).length,
        recent: this.errors.slice(-5)
      },
      slowQueries: {
        total: this.slowQueries.filter(q => q.timestamp.getTime() > last5Minutes).length,
        recent: this.slowQueries.slice(-5)
      }
    };
  }
  
  alertSlowTransaction(name, duration, metadata) {
    logger.error('Slow transaction detected', {
      transaction: name,
      duration: `${duration}ms`,
      metadata
    });
    
    // Send alert to Slack/PagerDuty
  }
  
  sendToSentry(error, context) {
    // Integration with Sentry
    console.error('Sending to Sentry:', error.message, context);
  }
}

/**
 * Health Check Endpoints
 */
class HealthCheckService {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
  }
  
  /**
   * Liveness probe (is app running?)
   */
  async liveness() {
    return {
      status: 'UP',
      timestamp: new Date().toISOString()
    };
  }
  
  /**
   * Readiness probe (is app ready to serve traffic?)
   */
  async readiness() {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis()
    ]);
    
    const dbCheck = checks[0];
    const redisCheck = checks[1];
    
    const isReady = dbCheck.status === 'fulfilled' && redisCheck.status === 'fulfilled';
    
    return {
      status: isReady ? 'UP' : 'DOWN',
      checks: {
        database: dbCheck.status === 'fulfilled' ? dbCheck.value : { status: 'DOWN', error: dbCheck.reason.message },
        redis: redisCheck.status === 'fulfilled' ? redisCheck.value : { status: 'DOWN', error: redisCheck.reason.message }
      },
      timestamp: new Date().toISOString()
    };
  }
  
  /**
   * Check database connectivity
   */
  async checkDatabase() {
    const start = Date.now();
    await this.db.query('SELECT 1');
    const duration = Date.now() - start;
    
    return {
      status: 'UP',
      responseTime: `${duration}ms`
    };
  }
  
  /**
   * Check Redis connectivity
   */
  async checkRedis() {
    const start = Date.now();
    await this.redis.ping();
    const duration = Date.now() - start;
    
    return {
      status: 'UP',
      responseTime: `${duration}ms`
    };
  }
}

/**
 * Express Routes
 */
const express = require('express');
const router = express.Router();

const metricsService = new MetricsService();
const tracingService = new TracingService();
const apmService = new APMService();
const healthCheckService = new HealthCheckService(db, redis);

// Metrics endpoint
router.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await metricsService.getMetrics());
});

// Health check endpoints
router.get('/health/liveness', async (req, res) => {
  const health = await healthCheckService.liveness();
  res.json(health);
});

router.get('/health/readiness', async (req, res) => {
  const health = await healthCheckService.readiness();
  const statusCode = health.status === 'UP' ? 200 : 503;
  res.status(statusCode).json(health);
});

// APM summary
router.get('/apm/summary', async (req, res) => {
  const summary = apmService.getSummary();
  res.json(summary);
});

module.exports = { 
  MetricsService, 
  TracingService, 
  APMService, 
  HealthCheckService,
  trackingMiddleware 
};
```

---

### Q14. How do you implement API testing strategies?

**Answer:**

```javascript
/**
 * Comprehensive API Testing with Jest and Supertest
 */
const request = require('supertest');
const { app } = require('../src/app');
const { db } = require('../src/db');
const jwt = require('jsonwebtoken');

/**
 * 1. Unit Tests for Business Logic
 */
describe('Customer Service Unit Tests', () => {
  const CustomerService = require('../src/services/customer.service');
  let customerService;
  let mockDb;
  
  beforeEach(() => {
    mockDb = {
      query: jest.fn()
    };
    customerService = new CustomerService(mockDb);
  });
  
  test('should create customer with valid data', async () => {
    const customerData = {
      emiratesId: '784-1234-1234567-1',
      name: 'Ahmed Al Maktoum',
      email: 'ahmed@example.com',
      phone: '+971-50-1234567'
    };
    
    mockDb.query.mockResolvedValue({
      rows: [{ id: '123', ...customerData }]
    });
    
    const result = await customerService.createCustomer(customerData);
    
    expect(result).toHaveProperty('id');
    expect(result.name).toBe(customerData.name);
    expect(mockDb.query).toHaveBeenCalledWith(
      expect.stringContaining('INSERT INTO customers'),
      expect.arrayContaining([customerData.emiratesId])
    );
  });
  
  test('should throw error for invalid Emirates ID', async () => {
    const customerData = {
      emiratesId: 'INVALID',
      name: 'Test',
      email: 'test@example.com'
    };
    
    await expect(customerService.createCustomer(customerData))
      .rejects
      .toThrow('Invalid Emirates ID format');
  });
});

/**
 * 2. Integration Tests
 */
describe('Customer API Integration Tests', () => {
  let authToken;
  let customerId;
  
  beforeAll(async () => {
    // Setup test database
    await db.query('BEGIN');
    
    // Create test user and get token
    const user = await db.query(`
      INSERT INTO customers (emirates_id, name, email, phone)
      VALUES ('784-1234-1234567-1', 'Test User', 'test@example.com', '+971-50-1234567')
      RETURNING *
    `);
    
    customerId = user.rows[0].id;
    
    authToken = jwt.sign(
      { userId: customerId, customerId, email: 'test@example.com' },
      process.env.JWT_SECRET,
      { expiresIn: '1h' }
    );
  });
  
  afterAll(async () => {
    // Rollback test data
    await db.query('ROLLBACK');
    await db.end();
  });
  
  describe('GET /api/customers/:id', () => {
    test('should return customer details', async () => {
      const response = await request(app)
        .get(`/api/customers/${customerId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(response.body).toHaveProperty('id', customerId);
      expect(response.body).toHaveProperty('name', 'Test User');
      expect(response.body).toHaveProperty('email', 'test@example.com');
    });
    
    test('should return 404 for non-existent customer', async () => {
      const response = await request(app)
        .get('/api/customers/00000000-0000-0000-0000-000000000000')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(404);
      
      expect(response.body).toHaveProperty('error', 'NotFound');
    });
    
    test('should return 401 without auth token', async () => {
      await request(app)
        .get(`/api/customers/${customerId}`)
        .expect(401);
    });
  });
  
  describe('POST /api/customers', () => {
    test('should create new customer', async () => {
      const newCustomer = {
        emiratesId: '784-2345-2345678-2',
        name: 'Mohammed Hassan',
        email: 'mohammed@example.com',
        phone: '+971-50-2345678',
        dateOfBirth: '1990-01-15',
        nationality: 'UAE'
      };
      
      const response = await request(app)
        .post('/api/customers')
        .set('Authorization', `Bearer ${authToken}`)
        .send(newCustomer)
        .expect(201);
      
      expect(response.body).toHaveProperty('id');
      expect(response.body.name).toBe(newCustomer.name);
      expect(response.headers.location).toContain('/api/customers/');
    });
    
    test('should return 400 for invalid data', async () => {
      const invalidCustomer = {
        emiratesId: 'INVALID',
        name: 'T', // Too short
        email: 'not-an-email'
      };
      
      const response = await request(app)
        .post('/api/customers')
        .set('Authorization', `Bearer ${authToken}`)
        .send(invalidCustomer)
        .expect(400);
      
      expect(response.body).toHaveProperty('error', 'ValidationError');
      expect(response.body.details).toHaveProperty('emiratesId');
    });
    
    test('should return 409 for duplicate Emirates ID', async () => {
      const duplicate = {
        emiratesId: '784-1234-1234567-1', // Already exists
        name: 'Duplicate',
        email: 'duplicate@example.com',
        phone: '+971-50-9999999'
      };
      
      const response = await request(app)
        .post('/api/customers')
        .set('Authorization', `Bearer ${authToken}`)
        .send(duplicate)
        .expect(409);
      
      expect(response.body).toHaveProperty('error', 'Conflict');
    });
  });
});

/**
 * 3. Contract Tests with Pact
 */
const { Pact } = require('@pact-foundation/pact');
const path = require('path');

describe('Customer API Contract Tests', () => {
  const provider = new Pact({
    consumer: 'WebApp',
    provider: 'CustomerAPI',
    port: 8989,
    log: path.resolve(process.cwd(), 'logs', 'pact.log'),
    dir: path.resolve(process.cwd(), 'pacts'),
    logLevel: 'INFO'
  });
  
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());
  
  test('should get customer by ID', async () => {
    await provider.addInteraction({
      state: 'customer exists',
      uponReceiving: 'a request for customer',
      withRequest: {
        method: 'GET',
        path: '/api/customers/123',
        headers: {
          Authorization: 'Bearer token123'
        }
      },
      willRespondWith: {
        status: 200,
        headers: {
          'Content-Type': 'application/json'
        },
        body: {
          id: '123',
          name: 'Ahmed Al Maktoum',
          email: 'ahmed@example.com'
        }
      }
    });
    
    const response = await request('http://localhost:8989')
      .get('/api/customers/123')
      .set('Authorization', 'Bearer token123');
    
    expect(response.status).toBe(200);
    expect(response.body.id).toBe('123');
  });
});

/**
 * 4. Load Tests with Artillery
 */
// artillery-config.yml
const artilleryConfig = `
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Ramp up load"
    - duration: 180
      arrivalRate: 100
      name: "Sustained load"
  plugins:
    expect: {}
  variables:
    authToken: "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

scenarios:
  - name: "Get customer accounts"
    flow:
      - get:
          url: "/api/customers/{{ $randomUUID }}/accounts"
          headers:
            Authorization: "{{ authToken }}"
          expect:
            - statusCode: 
                - 200
                - 404
      
  - name: "Create transaction"
    flow:
      - post:
          url: "/api/transactions"
          headers:
            Authorization: "{{ authToken }}"
            Content-Type: "application/json"
          json:
            accountId: "{{ $randomUUID }}"
            amount: "{{ $randomNumber(100, 10000) }}"
            type: "debit"
            description: "Load test transaction"
          expect:
            - statusCode: 
                - 201
                - 400
`;

/**
 * 5. E2E Tests with Playwright
 */
const { test, expect } = require('@playwright/test');

test.describe('Banking Portal E2E', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:3000');
  });
  
  test('should login and view accounts', async ({ page }) => {
    // Login
    await page.fill('input[name="email"]', 'ahmed@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    
    // Wait for redirect
    await page.waitForURL('**/dashboard');
    
    // Check accounts loaded
    await expect(page.locator('.account-card')).toHaveCount(2);
    
    // Click first account
    await page.click('.account-card:first-child');
    
    // Check transactions visible
    await expect(page.locator('.transaction-row')).toHaveCountGreaterThan(0);
  });
  
  test('should create new transaction', async ({ page }) => {
    // Login
    await page.fill('input[name="email"]', 'ahmed@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    
    await page.waitForURL('**/dashboard');
    
    // Navigate to transfer
    await page.click('text=Transfer');
    
    // Fill form
    await page.fill('input[name="amount"]', '500');
    await page.fill('input[name="beneficiaryIban"]', 'AE123456789012345678901');
    await page.fill('input[name="description"]', 'Test transfer');
    
    // Submit
    await page.click('button:has-text("Transfer")');
    
    // Check success
    await expect(page.locator('.success-message')).toBeVisible();
    await expect(page.locator('.success-message')).toContainText('Transfer successful');
  });
});

/**
 * 6. Snapshot Tests
 */
describe('API Response Snapshots', () => {
  test('customer response matches snapshot', async () => {
    const response = await request(app)
      .get('/api/customers/123')
      .set('Authorization', `Bearer ${authToken}`);
    
    expect(response.body).toMatchSnapshot({
      id: expect.any(String),
      createdAt: expect.any(String),
      updatedAt: expect.any(String)
    });
  });
});

module.exports = {
  // Export test utilities
};
```

---

**Summary Q12-Q14:**
- OAuth 2.0 implementation (Authorization Code, Client Credentials, PKCE, Refresh Token) ✅
- Comprehensive monitoring (Prometheus metrics, distributed tracing, APM, health checks) ✅
- Complete testing strategy (Unit, Integration, Contract, Load, E2E, Snapshot tests) ✅
