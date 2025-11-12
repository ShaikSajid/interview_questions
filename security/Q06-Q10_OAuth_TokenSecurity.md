# Security Questions 6-10: OAuth 2.0 & Token Security

---

### Q6. Explain OAuth 2.0 flows and implement Authorization Code flow

**Answer:**

OAuth 2.0 is an authorization framework that enables applications to obtain limited access to user accounts. It has four main flows (grant types):

**OAuth 2.0 Flows:**

1. **Authorization Code Flow** - Most secure, for server-side apps
2. **Implicit Flow** - For browser-based apps (deprecated)
3. **Client Credentials Flow** - For machine-to-machine
4. **Resource Owner Password Credentials** - For trusted apps

**Authorization Code Flow Implementation:**

```javascript
const express = require('express');
const axios = require('axios');
const crypto = require('crypto');
const jwt = require('jsonwebtoken');

/**
 * OAuth 2.0 Authorization Server for ENBD
 */
class OAuth2AuthorizationServer {
  constructor() {
    this.clients = new Map(); // Registered OAuth clients
    this.authorizationCodes = new Map(); // Temporary codes
    this.accessTokens = new Map(); // Active tokens
    this.refreshTokens = new Map(); // Refresh tokens
    
    // Register a sample client
    this.registerClient({
      clientId: 'enbd-mobile-app',
      clientSecret: 'secret123',
      redirectUris: ['enbd://callback', 'http://localhost:3000/callback'],
      grantTypes: ['authorization_code', 'refresh_token'],
      scope: ['read:accounts', 'write:transactions', 'read:profile']
    });
  }
  
  /**
   * Step 1: Authorization Endpoint
   * User is redirected here to grant permission
   */
  async authorize(req, res) {
    const {
      response_type,
      client_id,
      redirect_uri,
      scope,
      state,
      code_challenge,        // PKCE
      code_challenge_method  // PKCE
    } = req.query;
    
    // 1. Validate client
    const client = this.clients.get(client_id);
    if (!client) {
      return res.status(400).json({ error: 'invalid_client' });
    }
    
    // 2. Validate redirect URI
    if (!client.redirectUris.includes(redirect_uri)) {
      return res.status(400).json({ error: 'invalid_redirect_uri' });
    }
    
    // 3. Validate response type
    if (response_type !== 'code') {
      return res.status(400).json({ error: 'unsupported_response_type' });
    }
    
    // 4. Show login page (simplified - in reality, show actual login form)
    // Assume user is already authenticated
    const userId = req.session?.userId || 'user-123';
    
    // 5. Show consent page (simplified)
    const requestedScopes = scope ? scope.split(' ') : [];
    
    // User approves - generate authorization code
    const code = crypto.randomBytes(32).toString('hex');
    
    this.authorizationCodes.set(code, {
      clientId: client_id,
      userId,
      redirectUri: redirect_uri,
      scope: requestedScopes,
      codeChallenge: code_challenge,
      codeChallengeMethod: code_challenge_method,
      expiresAt: Date.now() + 10 * 60 * 1000, // 10 minutes
      used: false
    });
    
    // 6. Redirect back to client with code
    const redirectUrl = new URL(redirect_uri);
    redirectUrl.searchParams.append('code', code);
    if (state) {
      redirectUrl.searchParams.append('state', state);
    }
    
    res.redirect(redirectUrl.toString());
  }
  
  /**
   * Step 2: Token Endpoint
   * Exchange authorization code for tokens
   */
  async token(req, res) {
    const {
      grant_type,
      code,
      redirect_uri,
      client_id,
      client_secret,
      code_verifier,  // PKCE
      refresh_token
    } = req.body;
    
    // Handle different grant types
    if (grant_type === 'authorization_code') {
      return this.handleAuthorizationCodeGrant(req, res);
    } else if (grant_type === 'refresh_token') {
      return this.handleRefreshTokenGrant(req, res);
    } else {
      return res.status(400).json({ error: 'unsupported_grant_type' });
    }
  }
  
  async handleAuthorizationCodeGrant(req, res) {
    const {
      code,
      redirect_uri,
      client_id,
      client_secret,
      code_verifier
    } = req.body;
    
    // 1. Validate client credentials
    const client = this.clients.get(client_id);
    if (!client || client.clientSecret !== client_secret) {
      return res.status(401).json({ error: 'invalid_client' });
    }
    
    // 2. Validate authorization code
    const authCode = this.authorizationCodes.get(code);
    if (!authCode) {
      return res.status(400).json({ error: 'invalid_grant' });
    }
    
    // 3. Check if code is expired
    if (Date.now() > authCode.expiresAt) {
      this.authorizationCodes.delete(code);
      return res.status(400).json({ error: 'expired_grant' });
    }
    
    // 4. Check if code was already used
    if (authCode.used) {
      this.authorizationCodes.delete(code);
      return res.status(400).json({ error: 'invalid_grant' });
    }
    
    // 5. Validate redirect URI
    if (authCode.redirectUri !== redirect_uri) {
      return res.status(400).json({ error: 'invalid_grant' });
    }
    
    // 6. Validate PKCE if used
    if (authCode.codeChallenge) {
      if (!code_verifier) {
        return res.status(400).json({ error: 'invalid_request' });
      }
      
      const valid = this.validatePKCE(
        code_verifier,
        authCode.codeChallenge,
        authCode.codeChallengeMethod
      );
      
      if (!valid) {
        return res.status(400).json({ error: 'invalid_grant' });
      }
    }
    
    // 7. Mark code as used
    authCode.used = true;
    
    // 8. Generate tokens
    const accessToken = this.generateAccessToken({
      userId: authCode.userId,
      clientId: authCode.clientId,
      scope: authCode.scope
    });
    
    const refreshToken = this.generateRefreshToken({
      userId: authCode.userId,
      clientId: authCode.clientId,
      scope: authCode.scope
    });
    
    // 9. Store tokens
    this.accessTokens.set(accessToken.token, accessToken);
    this.refreshTokens.set(refreshToken.token, refreshToken);
    
    // 10. Return tokens
    res.json({
      access_token: accessToken.token,
      token_type: 'Bearer',
      expires_in: 3600,
      refresh_token: refreshToken.token,
      scope: authCode.scope.join(' ')
    });
    
    // Clean up authorization code
    setTimeout(() => {
      this.authorizationCodes.delete(code);
    }, 1000);
  }
  
  async handleRefreshTokenGrant(req, res) {
    const { refresh_token, client_id, client_secret } = req.body;
    
    // 1. Validate client
    const client = this.clients.get(client_id);
    if (!client || client.clientSecret !== client_secret) {
      return res.status(401).json({ error: 'invalid_client' });
    }
    
    // 2. Validate refresh token
    const tokenData = this.refreshTokens.get(refresh_token);
    if (!tokenData) {
      return res.status(400).json({ error: 'invalid_grant' });
    }
    
    // 3. Check expiration
    if (Date.now() > tokenData.expiresAt) {
      this.refreshTokens.delete(refresh_token);
      return res.status(400).json({ error: 'expired_token' });
    }
    
    // 4. Generate new access token
    const accessToken = this.generateAccessToken({
      userId: tokenData.userId,
      clientId: tokenData.clientId,
      scope: tokenData.scope
    });
    
    this.accessTokens.set(accessToken.token, accessToken);
    
    res.json({
      access_token: accessToken.token,
      token_type: 'Bearer',
      expires_in: 3600,
      scope: tokenData.scope.join(' ')
    });
  }
  
  generateAccessToken(data) {
    const token = jwt.sign(
      {
        sub: data.userId,
        client_id: data.clientId,
        scope: data.scope,
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + 3600
      },
      process.env.JWT_SECRET,
      { algorithm: 'HS256' }
    );
    
    return {
      token,
      userId: data.userId,
      clientId: data.clientId,
      scope: data.scope,
      expiresAt: Date.now() + 3600 * 1000
    };
  }
  
  generateRefreshToken(data) {
    const token = crypto.randomBytes(32).toString('hex');
    
    return {
      token,
      userId: data.userId,
      clientId: data.clientId,
      scope: data.scope,
      expiresAt: Date.now() + 30 * 24 * 60 * 60 * 1000 // 30 days
    };
  }
  
  validatePKCE(verifier, challenge, method) {
    if (method === 'S256') {
      const hash = crypto.createHash('sha256').update(verifier).digest('base64');
      const computed = hash.replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
      return computed === challenge;
    }
    return verifier === challenge;
  }
  
  registerClient(clientData) {
    this.clients.set(clientData.clientId, clientData);
  }
}

/**
 * OAuth 2.0 Client Implementation
 */
class OAuth2Client {
  constructor(config) {
    this.clientId = config.clientId;
    this.clientSecret = config.clientSecret;
    this.redirectUri = config.redirectUri;
    this.authorizationEndpoint = config.authorizationEndpoint;
    this.tokenEndpoint = config.tokenEndpoint;
    this.scope = config.scope;
    
    // PKCE
    this.codeVerifier = null;
    this.codeChallenge = null;
  }
  
  /**
   * Step 1: Generate authorization URL
   */
  getAuthorizationUrl(state) {
    // Generate PKCE challenge
    this.codeVerifier = this.generateCodeVerifier();
    this.codeChallenge = this.generateCodeChallenge(this.codeVerifier);
    
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.clientId,
      redirect_uri: this.redirectUri,
      scope: this.scope.join(' '),
      state: state || crypto.randomBytes(16).toString('hex'),
      code_challenge: this.codeChallenge,
      code_challenge_method: 'S256'
    });
    
    return `${this.authorizationEndpoint}?${params.toString()}`;
  }
  
  /**
   * Step 2: Exchange code for tokens
   */
  async exchangeCodeForTokens(code) {
    try {
      const response = await axios.post(this.tokenEndpoint, {
        grant_type: 'authorization_code',
        code,
        redirect_uri: this.redirectUri,
        client_id: this.clientId,
        client_secret: this.clientSecret,
        code_verifier: this.codeVerifier
      }, {
        headers: { 'Content-Type': 'application/json' }
      });
      
      return {
        success: true,
        ...response.data
      };
    } catch (error) {
      return {
        success: false,
        error: error.response?.data || error.message
      };
    }
  }
  
  /**
   * Refresh access token
   */
  async refreshAccessToken(refreshToken) {
    try {
      const response = await axios.post(this.tokenEndpoint, {
        grant_type: 'refresh_token',
        refresh_token: refreshToken,
        client_id: this.clientId,
        client_secret: this.clientSecret
      }, {
        headers: { 'Content-Type': 'application/json' }
      });
      
      return {
        success: true,
        ...response.data
      };
    } catch (error) {
      return {
        success: false,
        error: error.response?.data || error.message
      };
    }
  }
  
  generateCodeVerifier() {
    return crypto.randomBytes(32).toString('base64url');
  }
  
  generateCodeChallenge(verifier) {
    return crypto.createHash('sha256')
      .update(verifier)
      .digest('base64url');
  }
}

module.exports = { OAuth2AuthorizationServer, OAuth2Client };
```

**ENBD Banking OAuth Flow:**

```javascript
// 1. User clicks "Login with ENBD" on third-party app
const client = new OAuth2Client({
  clientId: 'enbd-mobile-app',
  clientSecret: 'secret123',
  redirectUri: 'http://localhost:3000/callback',
  authorizationEndpoint: 'https://auth.enbd.com/oauth/authorize',
  tokenEndpoint: 'https://auth.enbd.com/oauth/token',
  scope: ['read:accounts', 'read:transactions']
});

const authUrl = client.getAuthorizationUrl('random-state-123');
// Redirect user to: https://auth.enbd.com/oauth/authorize?response_type=code&client_id=enbd-mobile-app...

// 2. User logs in and approves permissions
// 3. User redirected back with code
// http://localhost:3000/callback?code=abc123&state=random-state-123

// 4. Exchange code for tokens
const tokens = await client.exchangeCodeForTokens('abc123');
/*
{
  access_token: "eyJhbGc...",
  token_type: "Bearer",
  expires_in: 3600,
  refresh_token: "def456",
  scope: "read:accounts read:transactions"
}
*/

// 5. Use access token to call API
const response = await axios.get('https://api.enbd.com/accounts', {
  headers: {
    'Authorization': `Bearer ${tokens.access_token}`
  }
});

// 6. When access token expires, refresh it
const newTokens = await client.refreshAccessToken(tokens.refresh_token);
```

---

### Q7. How do you implement Client Credentials flow for machine-to-machine authentication?

**Answer:**

```javascript
/**
 * Client Credentials Flow (Machine-to-Machine)
 */
class ClientCredentialsFlow {
  constructor() {
    this.clients = new Map();
    this.accessTokens = new Map();
  }
  
  /**
   * Register M2M client
   */
  registerClient(clientData) {
    const clientId = `client_${crypto.randomBytes(16).toString('hex')}`;
    const clientSecret = crypto.randomBytes(32).toString('hex');
    
    this.clients.set(clientId, {
      clientId,
      clientSecret,
      name: clientData.name,
      grantTypes: ['client_credentials'],
      scope: clientData.scope || [],
      createdAt: new Date()
    });
    
    return { clientId, clientSecret };
  }
  
  /**
   * Token endpoint for client credentials
   */
  async token(req, res) {
    const { grant_type, scope } = req.body;
    
    if (grant_type !== 'client_credentials') {
      return res.status(400).json({
        error: 'unsupported_grant_type'
      });
    }
    
    // Extract client credentials from Authorization header
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Basic ')) {
      return res.status(401).json({
        error: 'invalid_client',
        error_description: 'Client authentication required'
      });
    }
    
    // Decode Basic Auth
    const credentials = Buffer.from(
      authHeader.substring(6),
      'base64'
    ).toString('utf-8');
    
    const [clientId, clientSecret] = credentials.split(':');
    
    // Validate client
    const client = this.clients.get(clientId);
    if (!client || client.clientSecret !== clientSecret) {
      return res.status(401).json({
        error: 'invalid_client'
      });
    }
    
    // Validate requested scope
    const requestedScopes = scope ? scope.split(' ') : client.scope;
    const validScopes = requestedScopes.every(s => client.scope.includes(s));
    
    if (!validScopes) {
      return res.status(400).json({
        error: 'invalid_scope'
      });
    }
    
    // Generate access token
    const token = jwt.sign(
      {
        sub: clientId,
        client_id: clientId,
        scope: requestedScopes,
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + 3600,
        aud: 'enbd-api'
      },
      process.env.JWT_SECRET,
      { algorithm: 'HS256' }
    );
    
    // Store token
    this.accessTokens.set(token, {
      clientId,
      scope: requestedScopes,
      expiresAt: Date.now() + 3600 * 1000
    });
    
    res.json({
      access_token: token,
      token_type: 'Bearer',
      expires_in: 3600,
      scope: requestedScopes.join(' ')
    });
  }
}

/**
 * M2M Client for ENBD Microservices
 */
class ENBDServiceClient {
  constructor(clientId, clientSecret, tokenEndpoint) {
    this.clientId = clientId;
    this.clientSecret = clientSecret;
    this.tokenEndpoint = tokenEndpoint;
    this.cachedToken = null;
    this.tokenExpiry = null;
  }
  
  /**
   * Get access token (with caching)
   */
  async getAccessToken() {
    // Return cached token if still valid
    if (this.cachedToken && this.tokenExpiry > Date.now() + 60000) {
      return this.cachedToken;
    }
    
    // Request new token
    const credentials = Buffer.from(
      `${this.clientId}:${this.clientSecret}`
    ).toString('base64');
    
    try {
      const response = await axios.post(
        this.tokenEndpoint,
        { grant_type: 'client_credentials' },
        {
          headers: {
            'Authorization': `Basic ${credentials}`,
            'Content-Type': 'application/json'
          }
        }
      );
      
      this.cachedToken = response.data.access_token;
      this.tokenExpiry = Date.now() + (response.data.expires_in * 1000);
      
      return this.cachedToken;
    } catch (error) {
      throw new Error(`Failed to get access token: ${error.message}`);
    }
  }
  
  /**
   * Make authenticated API call
   */
  async callAPI(url, options = {}) {
    const token = await this.getAccessToken();
    
    return axios({
      url,
      ...options,
      headers: {
        ...options.headers,
        'Authorization': `Bearer ${token}`
      }
    });
  }
}

// Usage Example - Payment Service calling Transaction Service
class PaymentService {
  constructor() {
    this.transactionClient = new ENBDServiceClient(
      process.env.PAYMENT_SERVICE_CLIENT_ID,
      process.env.PAYMENT_SERVICE_CLIENT_SECRET,
      'https://auth.enbd.com/oauth/token'
    );
  }
  
  async processPayment(paymentData) {
    try {
      // Call transaction service
      const response = await this.transactionClient.callAPI(
        'https://api.enbd.com/internal/transactions',
        {
          method: 'POST',
          data: paymentData
        }
      );
      
      return response.data;
    } catch (error) {
      console.error('Payment failed:', error);
      throw error;
    }
  }
}
```

---

### Q8. How do you securely store JWT tokens on the client-side?

**Answer:**

```javascript
/**
 * Secure Token Storage Strategies
 */

// Strategy 1: HttpOnly Cookies (Most Secure for Web)
class SecureCookieStorage {
  /**
   * Server-side: Set tokens in HttpOnly cookies
   */
  static setTokens(res, accessToken, refreshToken) {
    // Access token in HttpOnly cookie
    res.cookie('access_token', accessToken, {
      httpOnly: true,        // Not accessible via JavaScript
      secure: true,          // Only sent over HTTPS
      sameSite: 'strict',    // CSRF protection
      maxAge: 15 * 60 * 1000 // 15 minutes
    });
    
    // Refresh token in separate HttpOnly cookie
    res.cookie('refresh_token', refreshToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
      path: '/api/auth/refresh' // Only sent to refresh endpoint
    });
  }
  
  /**
   * Server-side: Extract token from cookie
   */
  static extractToken(req) {
    return req.cookies.access_token;
  }
  
  /**
   * Server-side: Clear tokens
   */
  static clearTokens(res) {
    res.clearCookie('access_token');
    res.clearCookie('refresh_token');
  }
}

// Strategy 2: Memory Storage (Secure but lost on refresh)
class MemoryTokenStorage {
  constructor() {
    this.tokens = {
      accessToken: null,
      refreshToken: null,
      expiresAt: null
    };
  }
  
  setTokens(accessToken, refreshToken, expiresIn) {
    this.tokens = {
      accessToken,
      refreshToken,
      expiresAt: Date.now() + (expiresIn * 1000)
    };
  }
  
  getAccessToken() {
    if (this.tokens.expiresAt && Date.now() > this.tokens.expiresAt) {
      return null; // Expired
    }
    return this.tokens.accessToken;
  }
  
  getRefreshToken() {
    return this.tokens.refreshToken;
  }
  
  clear() {
    this.tokens = {
      accessToken: null,
      refreshToken: null,
      expiresAt: null
    };
  }
}

// Strategy 3: SessionStorage (Better than localStorage)
class SessionStorageManager {
  static setTokens(accessToken, refreshToken, expiresIn) {
    const data = {
      accessToken,
      refreshToken,
      expiresAt: Date.now() + (expiresIn * 1000)
    };
    
    sessionStorage.setItem('auth_tokens', JSON.stringify(data));
  }
  
  static getAccessToken() {
    const data = this.getTokenData();
    
    if (!data || Date.now() > data.expiresAt) {
      return null;
    }
    
    return data.accessToken;
  }
  
  static getRefreshToken() {
    const data = this.getTokenData();
    return data?.refreshToken;
  }
  
  static getTokenData() {
    const stored = sessionStorage.getItem('auth_tokens');
    return stored ? JSON.parse(stored) : null;
  }
  
  static clear() {
    sessionStorage.removeItem('auth_tokens');
  }
}

// Strategy 4: Encrypted LocalStorage (If must use localStorage)
class EncryptedLocalStorage {
  constructor(encryptionKey) {
    this.key = encryptionKey;
  }
  
  encrypt(data) {
    const crypto = window.crypto || window.msCrypto;
    // Use Web Crypto API for encryption
    // Simplified example - use proper encryption in production
    return btoa(JSON.stringify(data));
  }
  
  decrypt(encrypted) {
    try {
      return JSON.parse(atob(encrypted));
    } catch {
      return null;
    }
  }
  
  setTokens(accessToken, refreshToken, expiresIn) {
    const data = {
      accessToken,
      refreshToken,
      expiresAt: Date.now() + (expiresIn * 1000)
    };
    
    const encrypted = this.encrypt(data);
    localStorage.setItem('auth_data', encrypted);
  }
  
  getAccessToken() {
    const encrypted = localStorage.getItem('auth_data');
    if (!encrypted) return null;
    
    const data = this.decrypt(encrypted);
    
    if (!data || Date.now() > data.expiresAt) {
      this.clear();
      return null;
    }
    
    return data.accessToken;
  }
  
  clear() {
    localStorage.removeItem('auth_data');
  }
}

/**
 * Complete Authentication Service for ENBD Mobile App
 */
class ENBDAuthService {
  constructor() {
    // Use memory storage + HttpOnly cookies combination
    this.memoryStorage = new MemoryTokenStorage();
    this.apiClient = axios.create({
      baseURL: 'https://api.enbd.com',
      withCredentials: true // Send cookies
    });
    
    this.setupInterceptors();
  }
  
  setupInterceptors() {
    // Request interceptor - add token from memory
    this.apiClient.interceptors.request.use(
      (config) => {
        const token = this.memoryStorage.getAccessToken();
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );
    
    // Response interceptor - handle token refresh
    this.apiClient.interceptors.response.use(
      (response) => response,
      async (error) => {
        const originalRequest = error.config;
        
        // If 401 and haven't retried yet
        if (error.response?.status === 401 && !originalRequest._retry) {
          originalRequest._retry = true;
          
          try {
            // Refresh token
            await this.refreshToken();
            
            // Retry original request
            return this.apiClient(originalRequest);
          } catch (refreshError) {
            // Refresh failed - logout
            this.logout();
            return Promise.reject(refreshError);
          }
        }
        
        return Promise.reject(error);
      }
    );
  }
  
  async login(email, password) {
    try {
      const response = await this.apiClient.post('/auth/login', {
        email,
        password
      });
      
      const { accessToken, refreshToken, expiresIn } = response.data;
      
      // Store tokens in memory
      this.memoryStorage.setTokens(accessToken, refreshToken, expiresIn);
      
      // Tokens also stored in HttpOnly cookies by server
      
      return { success: true };
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
  
  async refreshToken() {
    const refreshToken = this.memoryStorage.getRefreshToken();
    
    const response = await this.apiClient.post('/auth/refresh', {
      refreshToken
    });
    
    const { accessToken, expiresIn } = response.data;
    
    this.memoryStorage.setTokens(
      accessToken,
      refreshToken,
      expiresIn
    );
  }
  
  async logout() {
    await this.apiClient.post('/auth/logout');
    this.memoryStorage.clear();
    window.location.href = '/login';
  }
  
  async getProfile() {
    const response = await this.apiClient.get('/profile');
    return response.data;
  }
}
```

**Security Comparison:**

| Storage Method | Security | Persistence | XSS Safe | CSRF Safe |
|----------------|----------|-------------|----------|-----------|
| HttpOnly Cookie | ⭐⭐⭐⭐⭐ | Yes | ✅ | ⚠️ Need SameSite |
| Memory | ⭐⭐⭐⭐ | No | ✅ | ✅ |
| SessionStorage | ⭐⭐⭐ | Tab only | ❌ | ✅ |
| LocalStorage | ⭐⭐ | Yes | ❌ | ✅ |

**Best Practice: HttpOnly Cookies + Memory**

---

### Q9. What are common JWT vulnerabilities and how to prevent them?

**Answer:**

```javascript
/**
 * JWT Security Vulnerabilities & Mitigations
 */

// Vulnerability 1: Algorithm Confusion Attack (alg: none)
class AlgorithmConfusionPrevention {
  /**
   * VULNERABLE CODE - Don't do this!
   */
  static vulnerableVerify(token) {
    const decoded = jwt.decode(token, { complete: true });
    
    // Attacker can set alg to "none"
    if (decoded.header.alg === 'none') {
      return decoded.payload; // No verification!
    }
    
    return jwt.verify(token, secret);
  }
  
  /**
   * SECURE CODE
   */
  static secureVerify(token, secret) {
    // Always specify allowed algorithms
    return jwt.verify(token, secret, {
      algorithms: ['HS256'], // Whitelist only
      // NEVER allow 'none'
    });
  }
  
  /**
   * Prevent algorithm switching (HS256 to RS256)
   */
  static verifyWithStrictAlgorithm(token, secret, expectedAlg) {
    try {
      const decoded = jwt.decode(token, { complete: true });
      
      // Verify algorithm matches expected
      if (decoded.header.alg !== expectedAlg) {
        throw new Error('Algorithm mismatch');
      }
      
      return jwt.verify(token, secret, {
        algorithms: [expectedAlg]
      });
    } catch (error) {
      throw new Error(`Token verification failed: ${error.message}`);
    }
  }
}

// Vulnerability 2: Weak Secrets
class WeakSecretPrevention {
  /**
   * Generate strong secret
   */
  static generateStrongSecret() {
    // Minimum 256 bits (32 bytes)
    return crypto.randomBytes(32).toString('hex');
  }
  
  /**
   * Validate secret strength
   */
  static validateSecret(secret) {
    // Check length
    if (secret.length < 32) {
      throw new Error('Secret too short. Minimum 32 characters.');
    }
    
    // Check entropy
    const entropy = this.calculateEntropy(secret);
    if (entropy < 4.0) {
      throw new Error('Secret has insufficient entropy');
    }
    
    return true;
  }
  
  static calculateEntropy(str) {
    const freq = {};
    for (let char of str) {
      freq[char] = (freq[char] || 0) + 1;
    }
    
    let entropy = 0;
    const len = str.length;
    
    for (let char in freq) {
      const p = freq[char] / len;
      entropy -= p * Math.log2(p);
    }
    
    return entropy;
  }
}

// Vulnerability 3: Token Exposure in URL
class TokenExposurePrevention {
  /**
   * VULNERABLE - Token in URL
   */
  static vulnerableAuth(req, res) {
    // Don't do this!
    const token = req.query.token;
    // URLs are logged, cached, visible in browser history
  }
  
  /**
   * SECURE - Token in Authorization header
   */
  static secureAuth(req, res) {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    const token = authHeader.substring(7);
    // Headers are not logged or cached
  }
}

// Vulnerability 4: Sensitive Data in Payload
class SensitiveDataPrevention {
  /**
   * VULNERABLE - Sensitive data in JWT
   */
  static vulnerableTokenGeneration(user) {
    return jwt.sign({
      id: user.id,
      password: user.password,        // NEVER include password!
      ssn: user.ssn,                  // NEVER include PII!
      accountBalance: user.balance    // Sensitive financial data
    }, secret);
  }
  
  /**
   * SECURE - Minimal data in JWT
   */
  static secureTokenGeneration(user) {
    return jwt.sign({
      sub: user.id,               // Only user identifier
      role: user.role,            // Role for authorization
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + 900
      // Fetch other data from database when needed
    }, secret);
  }
}

// Vulnerability 5: Missing Expiration
class ExpirationPrevention {
  /**
   * VULNERABLE - No expiration
   */
  static vulnerableToken(user) {
    return jwt.sign({ userId: user.id }, secret);
    // Token never expires!
  }
  
  /**
   * SECURE - Short expiration
   */
  static secureToken(user) {
    return jwt.sign(
      {
        sub: user.id,
        exp: Math.floor(Date.now() / 1000) + (15 * 60) // 15 minutes
      },
      secret
    );
  }
  
  /**
   * Check token expiration with buffer
   */
  static isTokenExpiringSoon(token, bufferSeconds = 60) {
    const decoded = jwt.decode(token);
    if (!decoded || !decoded.exp) return true;
    
    const expiresAt = decoded.exp * 1000;
    const threshold = Date.now() + (bufferSeconds * 1000);
    
    return expiresAt < threshold;
  }
}

// Vulnerability 6: Missing Token Validation
class ComprehensiveValidation {
  static validateToken(token) {
    try {
      // 1. Decode header to check structure
      const decoded = jwt.decode(token, { complete: true });
      
      if (!decoded) {
        throw new Error('Invalid token structure');
      }
      
      // 2. Check required header fields
      if (!decoded.header.alg || !decoded.header.typ) {
        throw new Error('Missing required header fields');
      }
      
      // 3. Verify algorithm is allowed
      const allowedAlgs = ['HS256', 'RS256'];
      if (!allowedAlgs.includes(decoded.header.alg)) {
        throw new Error('Invalid algorithm');
      }
      
      // 4. Verify signature
      const verified = jwt.verify(token, secret, {
        algorithms: [decoded.header.alg]
      });
      
      // 5. Check required claims
      const requiredClaims = ['sub', 'iat', 'exp'];
      for (let claim of requiredClaims) {
        if (!verified[claim]) {
          throw new Error(`Missing required claim: ${claim}`);
        }
      }
      
      // 6. Check issuer if specified
      if (verified.iss && verified.iss !== 'enbd-banking-api') {
        throw new Error('Invalid issuer');
      }
      
      // 7. Check audience if specified
      if (verified.aud && verified.aud !== 'enbd-web-app') {
        throw new Error('Invalid audience');
      }
      
      // 8. Check not-before time
      const now = Math.floor(Date.now() / 1000);
      if (verified.nbf && verified.nbf > now) {
        throw new Error('Token not yet valid');
      }
      
      return {
        valid: true,
        payload: verified
      };
      
    } catch (error) {
      return {
        valid: false,
        error: error.message
      };
    }
  }
}

// Vulnerability 7: JWT Injection
class InjectionPrevention {
  /**
   * Validate JWT format before processing
   */
  static isValidJWTFormat(token) {
    // JWT must have exactly 3 parts
    const parts = token.split('.');
    if (parts.length !== 3) {
      return false;
    }
    
    // Each part must be valid base64url
    const base64urlRegex = /^[A-Za-z0-9_-]+$/;
    return parts.every(part => base64urlRegex.test(part));
  }
  
  /**
   * Sanitize token input
   */
  static sanitizeToken(token) {
    // Remove any whitespace
    token = token.trim();
    
    // Remove Bearer prefix if present
    if (token.startsWith('Bearer ')) {
      token = token.substring(7);
    }
    
    // Validate format
    if (!this.isValidJWTFormat(token)) {
      throw new Error('Invalid JWT format');
    }
    
    return token;
  }
}

/**
 * Complete Secure JWT Service
 */
class SecureJWTService {
  constructor() {
    // Validate secret on initialization
    const secret = process.env.JWT_SECRET;
    WeakSecretPrevention.validateSecret(secret);
    this.secret = secret;
  }
  
  generateToken(user) {
    // Only include necessary data
    const payload = {
      sub: user.id,
      role: user.role,
      jti: crypto.randomUUID(),
      iss: 'enbd-banking-api',
      aud: 'enbd-web-app',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + 900,
      nbf: Math.floor(Date.now() / 1000)
    };
    
    // Use specific algorithm
    return jwt.sign(payload, this.secret, {
      algorithm: 'HS256'
    });
  }
  
  verifyToken(token) {
    // Sanitize input
    token = InjectionPrevention.sanitizeToken(token);
    
    // Comprehensive validation
    return ComprehensiveValidation.validateToken(token);
  }
}

module.exports = {
  AlgorithmConfusionPrevention,
  WeakSecretPrevention,
  SensitiveDataPrevention,
  ExpirationPrevention,
  ComprehensiveValidation,
  SecureJWTService
};
```

**Common Vulnerabilities Summary:**

1. ✅ Algorithm confusion (`alg: none`)
2. ✅ Weak secrets (< 256 bits)
3. ✅ Token in URLs
4. ✅ Sensitive data in payload
5. ✅ Missing/long expiration
6. ✅ Insufficient validation
7. ✅ JWT injection attacks

---

### Q10. How do you implement JWT with RSA (RS256) for public/private key signing?

**Answer:**

```javascript
const fs = require('fs');
const crypto = require('crypto');
const jwt = require('jsonwebtoken');
const { promisify } = require('util');

/**
 * RSA Key Management Service
 */
class RSAKeyManager {
  constructor() {
    this.keys = new Map();
    this.currentKeyId = null;
  }
  
  /**
   * Generate RSA key pair
   */
  async generateKeyPair(keyId) {
    const { privateKey, publicKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: {
        type: 'spki',
        format: 'pem'
      },
      privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem',
        cipher: 'aes-256-cbc',
        passphrase: process.env.KEY_PASSPHRASE
      }
    });
    
    this.keys.set(keyId, {
      keyId,
      privateKey,
      publicKey,
      createdAt: new Date(),
      algorithm: 'RS256'
    });
    
    this.currentKeyId = keyId;
    
    return { keyId, publicKey };
  }
  
  /**
   * Load keys from files
   */
  loadKeysFromFiles(privateKeyPath, publicKeyPath, keyId) {
    const privateKey = fs.readFileSync(privateKeyPath, 'utf8');
    const publicKey = fs.readFileSync(publicKeyPath, 'utf8');
    
    this.keys.set(keyId, {
      keyId,
      privateKey,
      publicKey,
      algorithm: 'RS256'
    });
    
    this.currentKeyId = keyId;
  }
  
  /**
   * Get private key for signing
   */
  getPrivateKey(keyId) {
    const key = this.keys.get(keyId || this.currentKeyId);
    if (!key) {
      throw new Error(`Key not found: ${keyId}`);
    }
    return key.privateKey;
  }
  
  /**
   * Get public key for verification
   */
  getPublicKey(keyId) {
    const key = this.keys.get(keyId || this.currentKeyId);
    if (!key) {
      throw new Error(`Key not found: ${keyId}`);
    }
    return key.publicKey;
  }
  
  /**
   * Get all public keys (for JWKS endpoint)
   */
  getPublicKeys() {
    const publicKeys = [];
    
    for (let [keyId, key] of this.keys) {
      publicKeys.push({
        kid: keyId,
        kty: 'RSA',
        use: 'sig',
        alg: 'RS256',
        n: this.extractModulus(key.publicKey),
        e: this.extractExponent(key.publicKey)
      });
    }
    
    return publicKeys;
  }
  
  extractModulus(publicKey) {
    // Extract modulus from public key
    // Simplified - use proper JWK conversion library
    return 'modulus_base64url';
  }
  
  extractExponent(publicKey) {
    // Extract exponent from public key
    return 'AQAB'; // Standard exponent
  }
  
  /**
   * Rotate keys
   */
  async rotateKeys() {
    const newKeyId = `key-${Date.now()}`;
    await this.generateKeyPair(newKeyId);
    
    // Keep old keys for verification (grace period)
    setTimeout(() => {
      // Remove old keys after 24 hours
      for (let [keyId, key] of this.keys) {
        if (keyId !== this.currentKeyId) {
          const age = Date.now() - key.createdAt.getTime();
          if (age > 24 * 60 * 60 * 1000) {
            this.keys.delete(keyId);
          }
        }
      }
    }, 24 * 60 * 60 * 1000);
  }
}

/**
 * RSA JWT Service (RS256)
 */
class RSAJWTService {
  constructor() {
    this.keyManager = new RSAKeyManager();
    
    // Generate initial key pair
    this.keyManager.generateKeyPair('key-initial');
    
    // Schedule key rotation (every 30 days)
    this.scheduleKeyRotation();
  }
  
  /**
   * Generate JWT with RSA signature
   */
  generateToken(user) {
    const payload = {
      sub: user.id,
      email: user.email,
      role: user.role,
      iss: 'enbd-banking-api',
      aud: 'enbd-web-app',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + 900,
      jti: crypto.randomUUID()
    };
    
    const privateKey = this.keyManager.getPrivateKey();
    
    const token = jwt.sign(payload, privateKey, {
      algorithm: 'RS256',
      keyid: this.keyManager.currentKeyId // Include key ID in header
    });
    
    return token;
  }
  
  /**
   * Verify JWT with RSA public key
   */
  verifyToken(token) {
    try {
      // Decode header to get key ID
      const decoded = jwt.decode(token, { complete: true });
      
      if (!decoded) {
        throw new Error('Invalid token');
      }
      
      const keyId = decoded.header.kid;
      const publicKey = this.keyManager.getPublicKey(keyId);
      
      // Verify with public key
      const verified = jwt.verify(token, publicKey, {
        algorithms: ['RS256'],
        issuer: 'enbd-banking-api',
        audience: 'enbd-web-app'
      });
      
      return {
        valid: true,
        payload: verified
      };
      
    } catch (error) {
      return {
        valid: false,
        error: error.message
      };
    }
  }
  
  /**
   * JWKS endpoint (JSON Web Key Set)
   * Public keys for external verification
   */
  getJWKS() {
    return {
      keys: this.keyManager.getPublicKeys()
    };
  }
  
  scheduleKeyRotation() {
    // Rotate keys every 30 days
    setInterval(() => {
      this.keyManager.rotateKeys();
    }, 30 * 24 * 60 * 60 * 1000);
  }
}

/**
 * Express Application with RSA JWT
 */
class RSAJWTApplication {
  constructor() {
    this.app = express();
    this.jwtService = new RSAJWTService();
    
    this.setupRoutes();
  }
  
  setupRoutes() {
    // JWKS endpoint - public keys for verification
    this.app.get('/.well-known/jwks.json', (req, res) => {
      res.json(this.jwtService.getJWKS());
    });
    
    // Login - generate token
    this.app.post('/api/auth/login', async (req, res) => {
      const { email, password } = req.body;
      
      // Authenticate user
      const user = await this.authenticateUser(email, password);
      
      if (!user) {
        return res.status(401).json({ error: 'Invalid credentials' });
      }
      
      // Generate RSA-signed token
      const token = this.jwtService.generateToken(user);
      
      res.json({
        accessToken: token,
        tokenType: 'Bearer',
        expiresIn: 900
      });
    });
    
    // Protected route
    this.app.get('/api/profile', this.authenticate.bind(this), (req, res) => {
      res.json(req.user);
    });
  }
  
  authenticate(req, res, next) {
    const authHeader = req.headers.authorization;
    
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    const token = authHeader.substring(7);
    const result = this.jwtService.verifyToken(token);
    
    if (!result.valid) {
      return res.status(401).json({ error: result.error });
    }
    
    req.user = result.payload;
    next();
  }
  
  async authenticateUser(email, password) {
    // Database lookup
    return {
      id: 'user-123',
      email,
      role: 'customer'
    };
  }
}

/**
 * External Service Verification (using JWKS)
 */
class ExternalServiceVerifier {
  constructor(jwksUrl) {
    this.jwksUrl = jwksUrl;
    this.publicKeys = new Map();
  }
  
  /**
   * Fetch public keys from JWKS endpoint
   */
  async fetchPublicKeys() {
    const response = await axios.get(this.jwksUrl);
    const jwks = response.data;
    
    for (let key of jwks.keys) {
      // Convert JWK to PEM format
      const publicKey = this.jwkToPem(key);
      this.publicKeys.set(key.kid, publicKey);
    }
  }
  
  /**
   * Verify token using fetched public keys
   */
  async verifyToken(token) {
    // Ensure we have public keys
    if (this.publicKeys.size === 0) {
      await this.fetchPublicKeys();
    }
    
    const decoded = jwt.decode(token, { complete: true });
    const keyId = decoded.header.kid;
    
    const publicKey = this.publicKeys.get(keyId);
    
    if (!publicKey) {
      // Refresh keys and try again
      await this.fetchPublicKeys();
      const publicKey = this.publicKeys.get(keyId);
      
      if (!publicKey) {
        throw new Error('Public key not found');
      }
    }
    
    return jwt.verify(token, publicKey, {
      algorithms: ['RS256']
    });
  }
  
  jwkToPem(jwk) {
    // Convert JWK to PEM format
    // Use library like jwk-to-pem
    return 'PEM_FORMAT_KEY';
  }
}

module.exports = { RSAKeyManager, RSAJWTService, RSAJWTApplication, ExternalServiceVerifier };
```

**ENBD Banking Usage:**

```javascript
// 1. Server generates RSA-signed token
const jwtService = new RSAJWTService();
const token = jwtService.generateToken(user);

// Token header includes key ID:
// {
//   "alg": "RS256",
//   "typ": "JWT",
//   "kid": "key-initial"
// }

// 2. External service fetches public keys
GET /.well-known/jwks.json
// Response:
{
  "keys": [
    {
      "kid": "key-initial",
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "n": "modulus...",
      "e": "AQAB"
    }
  ]
}

// 3. External service verifies token
const verifier = new ExternalServiceVerifier(
  'https://auth.enbd.com/.well-known/jwks.json'
);
const verified = await verifier.verifyToken(token);
```

**RS256 vs HS256:**

| Feature | HS256 | RS256 |
|---------|-------|-------|
| Algorithm | HMAC (Symmetric) | RSA (Asymmetric) |
| Key Distribution | Same secret everywhere | Public key can be shared |
| Performance | Faster | Slower |
| Use Case | Internal services | Public APIs, external clients |
| Key Rotation | Must update all services | Only update private key |
| Security | Good for internal | Better for external |

---

**Summary Q6-Q10:**
- OAuth 2.0 Authorization Code flow ✅
- Client Credentials flow (M2M) ✅
- Secure token storage strategies ✅
- JWT vulnerabilities & prevention ✅
- RSA (RS256) implementation with JWKS ✅

**Next batch: Q11-Q15 will cover advanced security topics...**
