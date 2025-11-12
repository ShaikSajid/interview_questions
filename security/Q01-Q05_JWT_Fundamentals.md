# Security Questions 1-5: JWT Fundamentals

---

### Q1. What is JWT and explain its structure in detail?

**Answer:**

JWT (JSON Web Token) is a compact, URL-safe token format used for securely transmitting information between parties as a JSON object. It's commonly used for authentication and information exchange in modern web applications.

**JWT Structure:**

A JWT consists of three parts separated by dots (.):
```
header.payload.signature
```

**1. Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
- `alg`: Algorithm used for signing (HS256, RS256, ES256)
- `typ`: Token type (always "JWT")

**2. Payload:**
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622,
  "role": "customer"
}
```
Contains claims (user data):
- **Registered claims**: `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`
- **Public claims**: Custom claims
- **Private claims**: Application-specific data

**3. Signature:**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

**Complete Implementation:**

```javascript
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

class JWTService {
  constructor() {
    // Use environment variable for secret
    this.secret = process.env.JWT_SECRET || crypto.randomBytes(64).toString('hex');
    this.refreshSecret = process.env.JWT_REFRESH_SECRET || crypto.randomBytes(64).toString('hex');
  }
  
  /**
   * Generate Access Token for ENBD Banking
   */
  generateAccessToken(user) {
    const payload = {
      // Registered claims
      sub: user.id,                    // Subject (user ID)
      iss: 'enbd-banking-api',         // Issuer
      aud: 'enbd-web-app',             // Audience
      iat: Math.floor(Date.now() / 1000),  // Issued at
      exp: Math.floor(Date.now() / 1000) + (15 * 60), // Expires in 15 minutes
      
      // Custom claims
      email: user.email,
      role: user.role,                 // 'customer', 'manager', 'admin'
      accountType: user.accountType,   // 'savings', 'current', 'premium'
      customerId: user.customerId,
      branch: user.branch,
      
      // Security claims
      tokenType: 'access',
      jti: crypto.randomUUID()         // JWT ID for tracking
    };
    
    return jwt.sign(payload, this.secret, {
      algorithm: 'HS256'
    });
  }
  
  /**
   * Generate Refresh Token
   */
  generateRefreshToken(user) {
    const payload = {
      sub: user.id,
      iss: 'enbd-banking-api',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + (7 * 24 * 60 * 60), // 7 days
      tokenType: 'refresh',
      jti: crypto.randomUUID()
    };
    
    return jwt.sign(payload, this.refreshSecret, {
      algorithm: 'HS256'
    });
  }
  
  /**
   * Verify and decode token
   */
  verifyToken(token, isRefreshToken = false) {
    try {
      const secret = isRefreshToken ? this.refreshSecret : this.secret;
      
      const decoded = jwt.verify(token, secret, {
        issuer: 'enbd-banking-api',
        audience: 'enbd-web-app',
        algorithms: ['HS256']
      });
      
      return {
        valid: true,
        decoded,
        expired: false
      };
    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        return {
          valid: false,
          expired: true,
          message: 'Token has expired'
        };
      }
      
      if (error.name === 'JsonWebTokenError') {
        return {
          valid: false,
          expired: false,
          message: 'Invalid token'
        };
      }
      
      return {
        valid: false,
        expired: false,
        message: error.message
      };
    }
  }
  
  /**
   * Decode without verification (for debugging)
   */
  decodeToken(token) {
    return jwt.decode(token, { complete: true });
  }
  
  /**
   * Get token expiry time
   */
  getTokenExpiry(token) {
    const decoded = jwt.decode(token);
    if (decoded && decoded.exp) {
      return new Date(decoded.exp * 1000);
    }
    return null;
  }
}

// Usage Example in Express
class BankingAuthController {
  constructor() {
    this.jwtService = new JWTService();
  }
  
  async login(req, res) {
    const { username, password } = req.body;
    
    try {
      // Authenticate user (check credentials)
      const user = await this.authenticateUser(username, password);
      
      if (!user) {
        return res.status(401).json({
          error: 'Invalid credentials'
        });
      }
      
      // Generate tokens
      const accessToken = this.jwtService.generateAccessToken(user);
      const refreshToken = this.jwtService.generateRefreshToken(user);
      
      // Store refresh token in database
      await this.storeRefreshToken(user.id, refreshToken);
      
      // Return tokens
      res.json({
        accessToken,
        refreshToken,
        tokenType: 'Bearer',
        expiresIn: 900, // 15 minutes in seconds
        user: {
          id: user.id,
          email: user.email,
          role: user.role
        }
      });
      
      // Log successful login
      await this.logAuditEvent('LOGIN_SUCCESS', user.id, req.ip);
      
    } catch (error) {
      console.error('Login error:', error);
      res.status(500).json({ error: 'Login failed' });
    }
  }
  
  async authenticateUser(username, password) {
    // Database lookup and password verification
    return {
      id: 'USR-123456',
      email: 'customer@enbd.com',
      role: 'customer',
      accountType: 'premium',
      customerId: 'CUST-789',
      branch: 'Dubai Main'
    };
  }
  
  async storeRefreshToken(userId, token) {
    // Store in Redis or database
  }
  
  async logAuditEvent(event, userId, ip) {
    // Audit logging
  }
}

module.exports = { JWTService, BankingAuthController };
```

**ENBD Banking Scenario:**

When a customer logs into ENBD mobile banking:

1. **Login Request:**
```javascript
POST /api/auth/login
{
  "username": "customer@enbd.com",
  "password": "SecurePass123!"
}
```

2. **Response with JWT:**
```javascript
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

3. **Decoded Access Token:**
```json
{
  "sub": "USR-123456",
  "iss": "enbd-banking-api",
  "aud": "enbd-web-app",
  "iat": 1699632000,
  "exp": 1699632900,
  "email": "customer@enbd.com",
  "role": "customer",
  "accountType": "premium",
  "customerId": "CUST-789",
  "branch": "Dubai Main",
  "tokenType": "access",
  "jti": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Best Practices:**

1. ✅ **Short expiry for access tokens** (15 minutes)
2. ✅ **Use strong secrets** (minimum 256 bits)
3. ✅ **Never store sensitive data** in payload
4. ✅ **Always use HTTPS**
5. ✅ **Implement token refresh mechanism**
6. ✅ **Use `jti` for token tracking/revocation**
7. ✅ **Log all token operations**
8. ✅ **Validate issuer and audience**

---

### Q2. How do you implement secure JWT token generation and signing?

**Answer:**

Secure JWT generation requires proper algorithm selection, key management, and payload design.

**Algorithm Selection:**

```javascript
const jwt = require('jsonwebtoken');
const fs = require('fs');
const crypto = require('crypto');

class SecureJWTService {
  constructor() {
    this.initializeKeys();
  }
  
  /**
   * Initialize cryptographic keys
   */
  initializeKeys() {
    // For HMAC (Symmetric - HS256)
    this.hmacSecret = process.env.JWT_SECRET || crypto.randomBytes(64).toString('hex');
    
    // For RSA (Asymmetric - RS256)
    if (process.env.JWT_PRIVATE_KEY_PATH) {
      this.rsaPrivateKey = fs.readFileSync(process.env.JWT_PRIVATE_KEY_PATH, 'utf8');
      this.rsaPublicKey = fs.readFileSync(process.env.JWT_PUBLIC_KEY_PATH, 'utf8');
    } else {
      // Generate RSA key pair for development
      const { privateKey, publicKey } = crypto.generateKeyPairSync('rsa', {
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
      
      this.rsaPrivateKey = privateKey;
      this.rsaPublicKey = publicKey;
    }
  }
  
  /**
   * Generate token with HMAC (HS256)
   * Best for: Internal microservices communication
   */
  generateTokenHMAC(payload) {
    const token = jwt.sign(
      {
        ...payload,
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + (15 * 60)
      },
      this.hmacSecret,
      {
        algorithm: 'HS256',
        issuer: 'enbd-banking-api',
        audience: 'enbd-services'
      }
    );
    
    return token;
  }
  
  /**
   * Generate token with RSA (RS256)
   * Best for: Public APIs, external clients
   */
  generateTokenRSA(payload) {
    const token = jwt.sign(
      {
        ...payload,
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + (15 * 60)
      },
      this.rsaPrivateKey,
      {
        algorithm: 'RS256',
        issuer: 'enbd-banking-api',
        audience: 'enbd-public-api',
        keyid: 'enbd-key-2024-11' // Key identifier for rotation
      }
    );
    
    return token;
  }
  
  /**
   * Verify HMAC token
   */
  verifyTokenHMAC(token) {
    try {
      const decoded = jwt.verify(token, this.hmacSecret, {
        algorithms: ['HS256'],
        issuer: 'enbd-banking-api',
        audience: 'enbd-services'
      });
      
      return { valid: true, decoded };
    } catch (error) {
      return { valid: false, error: error.message };
    }
  }
  
  /**
   * Verify RSA token
   */
  verifyTokenRSA(token) {
    try {
      const decoded = jwt.verify(token, this.rsaPublicKey, {
        algorithms: ['RS256'],
        issuer: 'enbd-banking-api',
        audience: 'enbd-public-api'
      });
      
      return { valid: true, decoded };
    } catch (error) {
      return { valid: false, error: error.message };
    }
  }
}

/**
 * Enhanced JWT Service with Security Features
 */
class EnhancedJWTService {
  constructor() {
    this.secret = process.env.JWT_SECRET;
    this.blacklistedTokens = new Set(); // In production, use Redis
  }
  
  /**
   * Generate secure token with all best practices
   */
  generateSecureToken(user, options = {}) {
    // Generate unique token ID
    const tokenId = crypto.randomUUID();
    
    // Create payload with minimal data
    const payload = {
      // Standard claims
      sub: user.id,
      iss: 'enbd-banking-api',
      aud: options.audience || 'enbd-web-app',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + (options.expiresIn || 900),
      nbf: Math.floor(Date.now() / 1000), // Not before
      jti: tokenId,
      
      // Custom claims (minimal)
      role: user.role,
      scope: options.scope || ['read:profile', 'write:transactions']
    };
    
    // Sign token
    const token = jwt.sign(payload, this.secret, {
      algorithm: 'HS256'
    });
    
    // Store token metadata (for revocation)
    this.storeTokenMetadata(tokenId, user.id, payload.exp);
    
    return {
      token,
      tokenId,
      expiresIn: options.expiresIn || 900,
      expiresAt: new Date(payload.exp * 1000)
    };
  }
  
  /**
   * Verify token with security checks
   */
  async verifySecureToken(token) {
    try {
      // 1. Verify signature and expiry
      const decoded = jwt.verify(token, this.secret, {
        algorithms: ['HS256'],
        issuer: 'enbd-banking-api'
      });
      
      // 2. Check if token is blacklisted
      if (this.blacklistedTokens.has(decoded.jti)) {
        throw new Error('Token has been revoked');
      }
      
      // 3. Check not-before time
      const now = Math.floor(Date.now() / 1000);
      if (decoded.nbf && decoded.nbf > now) {
        throw new Error('Token not yet valid');
      }
      
      // 4. Validate audience
      if (!this.validateAudience(decoded.aud)) {
        throw new Error('Invalid audience');
      }
      
      // 5. Check token metadata in database
      const isValid = await this.validateTokenMetadata(decoded.jti, decoded.sub);
      if (!isValid) {
        throw new Error('Token metadata invalid');
      }
      
      return {
        valid: true,
        decoded,
        userId: decoded.sub,
        role: decoded.role,
        scope: decoded.scope
      };
      
    } catch (error) {
      return {
        valid: false,
        error: error.message
      };
    }
  }
  
  /**
   * Revoke token (logout)
   */
  async revokeToken(token) {
    const decoded = jwt.decode(token);
    if (decoded && decoded.jti) {
      this.blacklistedTokens.add(decoded.jti);
      await this.removeTokenMetadata(decoded.jti);
    }
  }
  
  validateAudience(aud) {
    const validAudiences = ['enbd-web-app', 'enbd-mobile-app', 'enbd-public-api'];
    return validAudiences.includes(aud);
  }
  
  storeTokenMetadata(tokenId, userId, expiry) {
    // Store in Redis: SET token:{tokenId} userId EX {ttl}
  }
  
  async validateTokenMetadata(tokenId, userId) {
    // Check Redis: GET token:{tokenId}
    return true;
  }
  
  async removeTokenMetadata(tokenId) {
    // Remove from Redis: DEL token:{tokenId}
  }
}

module.exports = { SecureJWTService, EnhancedJWTService };
```

**ENBD Banking Scenario - Multiple Algorithms:**

```javascript
const express = require('express');
const app = express();

const secureJWT = new SecureJWTService();

// Internal microservice authentication (HS256)
app.post('/internal/api/transaction', async (req, res) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  const result = secureJWT.verifyTokenHMAC(token);
  
  if (!result.valid) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // Process internal transaction
  res.json({ success: true });
});

// Public API authentication (RS256)
app.post('/public/api/balance', async (req, res) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  const result = secureJWT.verifyTokenRSA(token);
  
  if (!result.valid) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // Return balance
  res.json({ balance: 10000 });
});
```

**Key Security Measures:**

1. ✅ **Algorithm Whitelisting**: Only allow specific algorithms
2. ✅ **Strong Secrets**: Minimum 256-bit for HMAC
3. ✅ **RSA 2048-bit**: For asymmetric signing
4. ✅ **Token ID (jti)**: Enables revocation
5. ✅ **Audience Validation**: Prevent token misuse
6. ✅ **Not-Before (nbf)**: Prevent premature use
7. ✅ **Metadata Storage**: Track active tokens
8. ✅ **Blacklist Support**: For logout/revocation

---

### Q3. How do you implement JWT middleware for authentication in Express?

**Answer:**

```javascript
const jwt = require('jsonwebtoken');

/**
 * JWT Authentication Middleware
 */
class JWTMiddleware {
  constructor(jwtService) {
    this.jwtService = jwtService;
  }
  
  /**
   * Basic authentication middleware
   */
  authenticate() {
    return async (req, res, next) => {
      try {
        // 1. Extract token from header
        const authHeader = req.headers.authorization;
        
        if (!authHeader) {
          return res.status(401).json({
            error: 'No authorization header'
          });
        }
        
        // 2. Parse Bearer token
        const parts = authHeader.split(' ');
        
        if (parts.length !== 2 || parts[0] !== 'Bearer') {
          return res.status(401).json({
            error: 'Invalid authorization format. Use: Bearer <token>'
          });
        }
        
        const token = parts[1];
        
        // 3. Verify token
        const result = await this.jwtService.verifySecureToken(token);
        
        if (!result.valid) {
          return res.status(401).json({
            error: 'Invalid or expired token',
            details: result.error
          });
        }
        
        // 4. Attach user info to request
        req.user = {
          id: result.decoded.sub,
          role: result.decoded.role,
          scope: result.decoded.scope,
          customerId: result.decoded.customerId
        };
        
        req.token = token;
        req.tokenId = result.decoded.jti;
        
        next();
        
      } catch (error) {
        console.error('Authentication error:', error);
        res.status(500).json({ error: 'Authentication failed' });
      }
    };
  }
  
  /**
   * Role-based authorization
   */
  authorize(...allowedRoles) {
    return (req, res, next) => {
      if (!req.user) {
        return res.status(401).json({
          error: 'Not authenticated'
        });
      }
      
      if (!allowedRoles.includes(req.user.role)) {
        return res.status(403).json({
          error: 'Insufficient permissions',
          required: allowedRoles,
          current: req.user.role
        });
      }
      
      next();
    };
  }
  
  /**
   * Scope-based authorization
   */
  requireScope(...requiredScopes) {
    return (req, res, next) => {
      if (!req.user || !req.user.scope) {
        return res.status(401).json({
          error: 'Not authenticated'
        });
      }
      
      const hasAllScopes = requiredScopes.every(scope =>
        req.user.scope.includes(scope)
      );
      
      if (!hasAllScopes) {
        return res.status(403).json({
          error: 'Insufficient scope',
          required: requiredScopes,
          current: req.user.scope
        });
      }
      
      next();
    };
  }
  
  /**
   * Optional authentication (doesn't fail if no token)
   */
  optionalAuth() {
    return async (req, res, next) => {
      const authHeader = req.headers.authorization;
      
      if (!authHeader) {
        return next();
      }
      
      try {
        const token = authHeader.split(' ')[1];
        const result = await this.jwtService.verifySecureToken(token);
        
        if (result.valid) {
          req.user = {
            id: result.decoded.sub,
            role: result.decoded.role
          };
        }
      } catch (error) {
        // Silently fail for optional auth
      }
      
      next();
    };
  }
}

/**
 * Complete Express Application with JWT
 */
class BankingApplication {
  constructor() {
    this.app = express();
    this.jwtService = new EnhancedJWTService();
    this.jwtMiddleware = new JWTMiddleware(this.jwtService);
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    
    // CORS
    this.app.use((req, res, next) => {
      res.header('Access-Control-Allow-Origin', process.env.ALLOWED_ORIGINS);
      res.header('Access-Control-Allow-Headers', 'Authorization, Content-Type');
      next();
    });
  }
  
  setupRoutes() {
    // Public routes (no authentication)
    this.app.post('/api/auth/login', this.login.bind(this));
    this.app.post('/api/auth/register', this.register.bind(this));
    
    // Protected routes (authentication required)
    this.app.get(
      '/api/profile',
      this.jwtMiddleware.authenticate(),
      this.getProfile.bind(this)
    );
    
    // Role-based protected routes
    this.app.get(
      '/api/accounts',
      this.jwtMiddleware.authenticate(),
      this.jwtMiddleware.authorize('customer', 'manager', 'admin'),
      this.getAccounts.bind(this)
    );
    
    // Admin only routes
    this.app.get(
      '/api/admin/users',
      this.jwtMiddleware.authenticate(),
      this.jwtMiddleware.authorize('admin'),
      this.getAllUsers.bind(this)
    );
    
    // Scope-based protected routes
    this.app.post(
      '/api/transactions',
      this.jwtMiddleware.authenticate(),
      this.jwtMiddleware.requireScope('write:transactions'),
      this.createTransaction.bind(this)
    );
    
    // Manager routes (multiple roles)
    this.app.get(
      '/api/reports',
      this.jwtMiddleware.authenticate(),
      this.jwtMiddleware.authorize('manager', 'admin'),
      this.getReports.bind(this)
    );
    
    // Optional auth route (public with enhanced features for authenticated)
    this.app.get(
      '/api/products',
      this.jwtMiddleware.optionalAuth(),
      this.getProducts.bind(this)
    );
    
    // Logout
    this.app.post(
      '/api/auth/logout',
      this.jwtMiddleware.authenticate(),
      this.logout.bind(this)
    );
  }
  
  async login(req, res) {
    const { email, password } = req.body;
    
    // Authenticate user
    const user = await this.authenticateUser(email, password);
    
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Generate token
    const { token, expiresIn } = this.jwtService.generateSecureToken(user, {
      audience: 'enbd-web-app',
      scope: this.getUserScope(user.role)
    });
    
    res.json({
      accessToken: token,
      tokenType: 'Bearer',
      expiresIn,
      user: {
        id: user.id,
        email: user.email,
        role: user.role
      }
    });
  }
  
  async getProfile(req, res) {
    // req.user is populated by authenticate middleware
    const profile = await this.fetchUserProfile(req.user.id);
    res.json(profile);
  }
  
  async getAccounts(req, res) {
    // Accessible by customer, manager, admin
    const accounts = await this.fetchAccounts(req.user.id);
    res.json(accounts);
  }
  
  async getAllUsers(req, res) {
    // Admin only
    const users = await this.fetchAllUsers();
    res.json(users);
  }
  
  async createTransaction(req, res) {
    // Requires 'write:transactions' scope
    const transaction = await this.processTransaction(req.body, req.user.id);
    res.json(transaction);
  }
  
  async getReports(req, res) {
    // Manager or Admin only
    const reports = await this.generateReports(req.user.role);
    res.json(reports);
  }
  
  async getProducts(req, res) {
    // Optional auth - enhanced response for authenticated users
    const products = await this.fetchProducts();
    
    if (req.user) {
      // Add personalized recommendations
      products.recommendations = await this.getPersonalizedProducts(req.user.id);
    }
    
    res.json(products);
  }
  
  async logout(req, res) {
    // Revoke token
    await this.jwtService.revokeToken(req.token);
    
    res.json({ message: 'Logged out successfully' });
  }
  
  getUserScope(role) {
    const scopes = {
      customer: ['read:profile', 'read:accounts', 'write:transactions'],
      manager: ['read:profile', 'read:accounts', 'write:transactions', 'read:reports'],
      admin: ['read:profile', 'read:accounts', 'write:transactions', 'read:reports', 'write:users']
    };
    
    return scopes[role] || [];
  }
  
  async authenticateUser(email, password) {
    // Database lookup
    return { id: 'USR-123', email, role: 'customer' };
  }
  
  async fetchUserProfile(userId) {
    return { id: userId, name: 'John Doe' };
  }
  
  async fetchAccounts(userId) {
    return [{ id: 'ACC-001', balance: 10000 }];
  }
  
  async fetchAllUsers() {
    return [];
  }
  
  async processTransaction(data, userId) {
    return { id: 'TXN-001', status: 'completed' };
  }
  
  async generateReports(role) {
    return [];
  }
  
  async fetchProducts() {
    return [];
  }
  
  async getPersonalizedProducts(userId) {
    return [];
  }
  
  async register(req, res) {
    res.json({ message: 'Registration endpoint' });
  }
}

module.exports = { JWTMiddleware, BankingApplication };
```

**ENBD Banking Usage Example:**

```javascript
// Customer accessing their profile
GET /api/profile
Headers: {
  "Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

// Response: 200 OK
{
  "id": "USR-123",
  "name": "Ahmed Al Mansoori",
  "email": "ahmed@enbd.com"
}

// Customer trying to access admin route
GET /api/admin/users
Headers: {
  "Authorization": "Bearer <customer-token>"
}

// Response: 403 Forbidden
{
  "error": "Insufficient permissions",
  "required": ["admin"],
  "current": "customer"
}
```

**Best Practices:**

1. ✅ **Consistent error responses**
2. ✅ **Role and scope separation**
3. ✅ **Optional authentication support**
4. ✅ **Request context enrichment**
5. ✅ **Token revocation on logout**
6. ✅ **Audit logging integration**

---

### Q4. How do you implement token refresh mechanism?

**Answer:**

```javascript
const crypto = require('crypto');
const jwt = require('jsonwebtoken');

/**
 * Token Refresh Service
 */
class TokenRefreshService {
  constructor() {
    this.accessTokenSecret = process.env.JWT_SECRET;
    this.refreshTokenSecret = process.env.JWT_REFRESH_SECRET;
    this.accessTokenExpiry = 15 * 60; // 15 minutes
    this.refreshTokenExpiry = 7 * 24 * 60 * 60; // 7 days
  }
  
  /**
   * Generate both access and refresh tokens
   */
  generateTokenPair(user) {
    const tokenId = crypto.randomUUID();
    const refreshTokenId = crypto.randomUUID();
    
    // Access Token (short-lived)
    const accessToken = jwt.sign(
      {
        sub: user.id,
        iss: 'enbd-banking-api',
        aud: 'enbd-web-app',
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + this.accessTokenExpiry,
        jti: tokenId,
        role: user.role,
        scope: user.scope,
        tokenType: 'access',
        refreshTokenId // Link to refresh token
      },
      this.accessTokenSecret,
      { algorithm: 'HS256' }
    );
    
    // Refresh Token (long-lived)
    const refreshToken = jwt.sign(
      {
        sub: user.id,
        iss: 'enbd-banking-api',
        aud: 'enbd-web-app',
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + this.refreshTokenExpiry,
        jti: refreshTokenId,
        tokenType: 'refresh',
        accessTokenId: tokenId // Link to access token
      },
      this.refreshTokenSecret,
      { algorithm: 'HS256' }
    );
    
    return {
      accessToken,
      refreshToken,
      accessTokenId: tokenId,
      refreshTokenId,
      expiresIn: this.accessTokenExpiry
    };
  }
  
  /**
   * Refresh access token using refresh token
   */
  async refreshAccessToken(refreshToken) {
    try {
      // 1. Verify refresh token
      const decoded = jwt.verify(refreshToken, this.refreshTokenSecret, {
        algorithms: ['HS256'],
        issuer: 'enbd-banking-api'
      });
      
      // 2. Check if refresh token is revoked
      const isRevoked = await this.isRefreshTokenRevoked(decoded.jti);
      if (isRevoked) {
        throw new Error('Refresh token has been revoked');
      }
      
      // 3. Get user data
      const user = await this.getUserById(decoded.sub);
      if (!user) {
        throw new Error('User not found');
      }
      
      // 4. Generate new access token
      const newAccessTokenId = crypto.randomUUID();
      
      const newAccessToken = jwt.sign(
        {
          sub: user.id,
          iss: 'enbd-banking-api',
          aud: 'enbd-web-app',
          iat: Math.floor(Date.now() / 1000),
          exp: Math.floor(Date.now() / 1000) + this.accessTokenExpiry,
          jti: newAccessTokenId,
          role: user.role,
          scope: user.scope,
          tokenType: 'access',
          refreshTokenId: decoded.jti
        },
        this.accessTokenSecret,
        { algorithm: 'HS256' }
      );
      
      // 5. Update refresh token usage
      await this.updateRefreshTokenUsage(decoded.jti, newAccessTokenId);
      
      return {
        success: true,
        accessToken: newAccessToken,
        tokenType: 'Bearer',
        expiresIn: this.accessTokenExpiry
      };
      
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  /**
   * Refresh token rotation (more secure)
   */
  async refreshWithRotation(refreshToken) {
    try {
      // 1. Verify current refresh token
      const decoded = jwt.verify(refreshToken, this.refreshTokenSecret, {
        algorithms: ['HS256']
      });
      
      // 2. Check for token reuse (security measure)
      const isReused = await this.checkTokenReuse(decoded.jti);
      if (isReused) {
        // Token reuse detected - revoke all user tokens
        await this.revokeAllUserTokens(decoded.sub);
        throw new Error('Token reuse detected. All tokens revoked.');
      }
      
      // 3. Get user
      const user = await this.getUserById(decoded.sub);
      
      // 4. Generate new token pair
      const newTokens = this.generateTokenPair(user);
      
      // 5. Revoke old refresh token
      await this.revokeRefreshToken(decoded.jti);
      
      // 6. Store new refresh token
      await this.storeRefreshToken(
        newTokens.refreshTokenId,
        user.id,
        this.refreshTokenExpiry
      );
      
      return {
        success: true,
        ...newTokens
      };
      
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  // Helper methods
  async isRefreshTokenRevoked(tokenId) {
    // Check Redis: EXISTS revoked:refresh:{tokenId}
    return false;
  }
  
  async getUserById(userId) {
    // Database lookup
    return {
      id: userId,
      role: 'customer',
      scope: ['read:profile', 'write:transactions']
    };
  }
  
  async updateRefreshTokenUsage(tokenId, newAccessTokenId) {
    // Update Redis: HSET refresh:{tokenId} lastUsed {timestamp} accessToken {newAccessTokenId}
  }
  
  async checkTokenReuse(tokenId) {
    // Check if token was already used to generate new tokens
    return false;
  }
  
  async revokeAllUserTokens(userId) {
    // Revoke all tokens for this user
  }
  
  async revokeRefreshToken(tokenId) {
    // Add to revoked list: SADD revoked:refresh:{tokenId}
  }
  
  async storeRefreshToken(tokenId, userId, expiry) {
    // Store in Redis: SET refresh:{tokenId} {userId} EX {expiry}
  }
}

/**
 * Express Implementation
 */
class TokenRefreshController {
  constructor() {
    this.refreshService = new TokenRefreshService();
  }
  
  setupRoutes(app) {
    // Login - returns both tokens
    app.post('/api/auth/login', this.login.bind(this));
    
    // Refresh - get new access token
    app.post('/api/auth/refresh', this.refresh.bind(this));
    
    // Refresh with rotation - get new token pair
    app.post('/api/auth/refresh-rotate', this.refreshRotate.bind(this));
    
    // Logout - revoke tokens
    app.post('/api/auth/logout', this.logout.bind(this));
  }
  
  async login(req, res) {
    const { email, password } = req.body;
    
    // Authenticate
    const user = await this.authenticateUser(email, password);
    
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Generate token pair
    const tokens = this.refreshService.generateTokenPair(user);
    
    // Store refresh token
    await this.refreshService.storeRefreshToken(
      tokens.refreshTokenId,
      user.id,
      this.refreshService.refreshTokenExpiry
    );
    
    res.json({
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
      tokenType: 'Bearer',
      expiresIn: tokens.expiresIn
    });
  }
  
  async refresh(req, res) {
    const { refreshToken } = req.body;
    
    if (!refreshToken) {
      return res.status(400).json({ error: 'Refresh token required' });
    }
    
    const result = await this.refreshService.refreshAccessToken(refreshToken);
    
    if (!result.success) {
      return res.status(401).json({ error: result.error });
    }
    
    res.json({
      accessToken: result.accessToken,
      tokenType: result.tokenType,
      expiresIn: result.expiresIn
    });
  }
  
  async refreshRotate(req, res) {
    const { refreshToken } = req.body;
    
    if (!refreshToken) {
      return res.status(400).json({ error: 'Refresh token required' });
    }
    
    const result = await this.refreshService.refreshWithRotation(refreshToken);
    
    if (!result.success) {
      return res.status(401).json({ error: result.error });
    }
    
    res.json({
      accessToken: result.accessToken,
      refreshToken: result.refreshToken,
      tokenType: 'Bearer',
      expiresIn: result.expiresIn
    });
  }
  
  async logout(req, res) {
    const { refreshToken } = req.body;
    
    if (refreshToken) {
      const decoded = jwt.decode(refreshToken);
      if (decoded && decoded.jti) {
        await this.refreshService.revokeRefreshToken(decoded.jti);
      }
    }
    
    res.json({ message: 'Logged out successfully' });
  }
  
  async authenticateUser(email, password) {
    return {
      id: 'USR-123',
      email,
      role: 'customer',
      scope: ['read:profile', 'write:transactions']
    };
  }
}

module.exports = { TokenRefreshService, TokenRefreshController };
```

**ENBD Banking Flow:**

```javascript
// 1. Initial Login
POST /api/auth/login
{
  "email": "customer@enbd.com",
  "password": "SecurePass123!"
}

// Response:
{
  "accessToken": "eyJhbGc...", // Expires in 15 min
  "refreshToken": "eyJhbGc...", // Expires in 7 days
  "tokenType": "Bearer",
  "expiresIn": 900
}

// 2. Access token expires after 15 minutes
// 3. Use refresh token to get new access token
POST /api/auth/refresh
{
  "refreshToken": "eyJhbGc..."
}

// Response:
{
  "accessToken": "eyJhbGc...", // New token
  "tokenType": "Bearer",
  "expiresIn": 900
}

// 4. With rotation (more secure)
POST /api/auth/refresh-rotate
{
  "refreshToken": "eyJhbGc..."
}

// Response:
{
  "accessToken": "eyJhbGc...", // New access token
  "refreshToken": "eyJhbGc...", // New refresh token
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

**Best Practices:**

1. ✅ **Short-lived access tokens** (15 minutes)
2. ✅ **Long-lived refresh tokens** (7 days)
3. ✅ **Token rotation** for enhanced security
4. ✅ **Reuse detection** to prevent attacks
5. ✅ **Revocation support** for logout
6. ✅ **Linked tokens** for tracking
7. ✅ **Secure storage** in Redis/database

---

### Q5. How do you implement JWT token revocation and blacklisting?

**Answer:**

```javascript
const Redis = require('ioredis');
const jwt = require('jsonwebtoken');

/**
 * Token Revocation Service using Redis
 */
class TokenRevocationService {
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379,
      password: process.env.REDIS_PASSWORD
    });
    
    this.secret = process.env.JWT_SECRET;
  }
  
  /**
   * Strategy 1: Blacklist individual tokens
   */
  async blacklistToken(token) {
    try {
      const decoded = jwt.decode(token);
      
      if (!decoded || !decoded.jti || !decoded.exp) {
        throw new Error('Invalid token format');
      }
      
      // Calculate TTL (time until expiration)
      const now = Math.floor(Date.now() / 1000);
      const ttl = decoded.exp - now;
      
      if (ttl <= 0) {
        // Token already expired, no need to blacklist
        return { success: true, message: 'Token already expired' };
      }
      
      // Add token ID to blacklist with expiration
      await this.redis.setex(
        `blacklist:token:${decoded.jti}`,
        ttl,
        JSON.stringify({
          userId: decoded.sub,
          revokedAt: new Date().toISOString(),
          reason: 'user_logout'
        })
      );
      
      return { success: true, message: 'Token blacklisted' };
      
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
  
  /**
   * Check if token is blacklisted
   */
  async isTokenBlacklisted(tokenId) {
    const result = await this.redis.get(`blacklist:token:${tokenId}`);
    return result !== null;
  }
  
  /**
   * Strategy 2: Revoke all tokens for a user
   */
  async revokeAllUserTokens(userId, reason = 'security') {
    try {
      // Set user revocation timestamp
      const revocationTime = Date.now();
      
      await this.redis.set(
        `revoke:user:${userId}`,
        revocationTime.toString(),
        'EX',
        30 * 24 * 60 * 60 // 30 days
      );
      
      // Log revocation event
      await this.logRevocation(userId, reason, 'all_tokens');
      
      return { success: true, revocationTime };
      
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
  
  /**
   * Check if user's tokens are revoked
   */
  async isUserTokenRevoked(userId, tokenIssuedAt) {
    const revocationTime = await this.redis.get(`revoke:user:${userId}`);
    
    if (!revocationTime) {
      return false;
    }
    
    // Token is revoked if issued before revocation time
    return tokenIssuedAt < parseInt(revocationTime) / 1000;
  }
  
  /**
   * Strategy 3: Token family revocation (refresh token chain)
   */
  async revokeTokenFamily(refreshTokenId) {
    try {
      // Get all tokens in the family
      const familyKey = `token:family:${refreshTokenId}`;
      const tokens = await this.redis.smembers(familyKey);
      
      // Blacklist all tokens in the family
      const pipeline = this.redis.pipeline();
      
      for (const tokenId of tokens) {
        pipeline.sadd('blacklist:tokens', tokenId);
      }
      
      pipeline.del(familyKey);
      await pipeline.exec();
      
      return { success: true, revokedCount: tokens.length };
      
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
  
  /**
   * Strategy 4: Session-based revocation
   */
  async createSession(userId, tokenId, deviceInfo) {
    const sessionId = require('crypto').randomUUID();
    
    const session = {
      sessionId,
      userId,
      tokenId,
      deviceInfo,
      createdAt: new Date().toISOString(),
      lastActivity: new Date().toISOString()
    };
    
    // Store session
    await this.redis.setex(
      `session:${sessionId}`,
      7 * 24 * 60 * 60, // 7 days
      JSON.stringify(session)
    );
    
    // Map token to session
    await this.redis.setex(
      `token:session:${tokenId}`,
      7 * 24 * 60 * 60,
      sessionId
    );
    
    // Add to user's active sessions
    await this.redis.sadd(`user:sessions:${userId}`, sessionId);
    
    return sessionId;
  }
  
  /**
   * Revoke session
   */
  async revokeSession(sessionId) {
    const sessionData = await this.redis.get(`session:${sessionId}`);
    
    if (!sessionData) {
      return { success: false, error: 'Session not found' };
    }
    
    const session = JSON.parse(sessionData);
    
    // Blacklist token
    await this.blacklistToken(session.tokenId);
    
    // Remove session
    await this.redis.del(`session:${sessionId}`);
    await this.redis.del(`token:session:${session.tokenId}`);
    await this.redis.srem(`user:sessions:${session.userId}`, sessionId);
    
    return { success: true };
  }
  
  /**
   * Get all active sessions for user
   */
  async getUserSessions(userId) {
    const sessionIds = await this.redis.smembers(`user:sessions:${userId}`);
    
    const sessions = [];
    for (const sessionId of sessionIds) {
      const data = await this.redis.get(`session:${sessionId}`);
      if (data) {
        sessions.push(JSON.parse(data));
      }
    }
    
    return sessions;
  }
  
  /**
   * Revoke all user sessions except current
   */
  async revokeOtherSessions(userId, currentSessionId) {
    const sessions = await this.getUserSessions(userId);
    
    let revokedCount = 0;
    for (const session of sessions) {
      if (session.sessionId !== currentSessionId) {
        await this.revokeSession(session.sessionId);
        revokedCount++;
      }
    }
    
    return { success: true, revokedCount };
  }
  
  async logRevocation(userId, reason, scope) {
    // Log to audit trail
    console.log(`Token revocation: ${userId}, reason: ${reason}, scope: ${scope}`);
  }
}

/**
 * Enhanced JWT Middleware with Revocation Check
 */
class SecureJWTMiddleware {
  constructor(revocationService) {
    this.revocationService = revocationService;
    this.secret = process.env.JWT_SECRET;
  }
  
  authenticate() {
    return async (req, res, next) => {
      try {
        const token = req.headers.authorization?.split(' ')[1];
        
        if (!token) {
          return res.status(401).json({ error: 'No token provided' });
        }
        
        // 1. Verify JWT signature and expiry
        const decoded = jwt.verify(token, this.secret, {
          algorithms: ['HS256']
        });
        
        // 2. Check if token is blacklisted
        const isBlacklisted = await this.revocationService.isTokenBlacklisted(decoded.jti);
        
        if (isBlacklisted) {
          return res.status(401).json({
            error: 'Token has been revoked',
            code: 'TOKEN_REVOKED'
          });
        }
        
        // 3. Check if all user tokens are revoked
        const isUserRevoked = await this.revocationService.isUserTokenRevoked(
          decoded.sub,
          decoded.iat
        );
        
        if (isUserRevoked) {
          return res.status(401).json({
            error: 'All user tokens have been revoked',
            code: 'USER_TOKENS_REVOKED'
          });
        }
        
        // 4. Verify session is still active
        const sessionId = await this.revocationService.redis.get(
          `token:session:${decoded.jti}`
        );
        
        if (!sessionId) {
          return res.status(401).json({
            error: 'Session not found',
            code: 'SESSION_INVALID'
          });
        }
        
        // Attach user info
        req.user = {
          id: decoded.sub,
          role: decoded.role,
          sessionId
        };
        
        req.token = token;
        req.tokenId = decoded.jti;
        
        next();
        
      } catch (error) {
        if (error.name === 'TokenExpiredError') {
          return res.status(401).json({
            error: 'Token expired',
            code: 'TOKEN_EXPIRED'
          });
        }
        
        res.status(401).json({ error: 'Invalid token' });
      }
    };
  }
}

/**
 * Express Routes for Token Management
 */
class TokenManagementController {
  constructor() {
    this.revocationService = new TokenRevocationService();
    this.middleware = new SecureJWTMiddleware(this.revocationService);
  }
  
  setupRoutes(app) {
    // Logout (revoke current token)
    app.post(
      '/api/auth/logout',
      this.middleware.authenticate(),
      this.logout.bind(this)
    );
    
    // Logout all devices (revoke all user tokens)
    app.post(
      '/api/auth/logout-all',
      this.middleware.authenticate(),
      this.logoutAll.bind(this)
    );
    
    // Get active sessions
    app.get(
      '/api/auth/sessions',
      this.middleware.authenticate(),
      this.getSessions.bind(this)
    );
    
    // Revoke specific session
    app.delete(
      '/api/auth/sessions/:sessionId',
      this.middleware.authenticate(),
      this.revokeSession.bind(this)
    );
    
    // Logout other devices
    app.post(
      '/api/auth/logout-others',
      this.middleware.authenticate(),
      this.logoutOthers.bind(this)
    );
  }
  
  async logout(req, res) {
    // Revoke current token
    await this.revocationService.blacklistToken(req.token);
    
    // Revoke session
    if (req.user.sessionId) {
      await this.revocationService.revokeSession(req.user.sessionId);
    }
    
    res.json({ message: 'Logged out successfully' });
  }
  
  async logoutAll(req, res) {
    // Revoke all user tokens
    await this.revocationService.revokeAllUserTokens(
      req.user.id,
      'user_requested'
    );
    
    res.json({ message: 'Logged out from all devices' });
  }
  
  async getSessions(req, res) {
    const sessions = await this.revocationService.getUserSessions(req.user.id);
    
    res.json({
      sessions: sessions.map(s => ({
        sessionId: s.sessionId,
        deviceInfo: s.deviceInfo,
        createdAt: s.createdAt,
        lastActivity: s.lastActivity,
        current: s.sessionId === req.user.sessionId
      }))
    });
  }
  
  async revokeSession(req, res) {
    const { sessionId } = req.params;
    
    // Verify session belongs to user
    const sessions = await this.revocationService.getUserSessions(req.user.id);
    const session = sessions.find(s => s.sessionId === sessionId);
    
    if (!session) {
      return res.status(404).json({ error: 'Session not found' });
    }
    
    await this.revocationService.revokeSession(sessionId);
    
    res.json({ message: 'Session revoked' });
  }
  
  async logoutOthers(req, res) {
    const result = await this.revocationService.revokeOtherSessions(
      req.user.id,
      req.user.sessionId
    );
    
    res.json({
      message: 'Other sessions revoked',
      revokedCount: result.revokedCount
    });
  }
}

module.exports = { TokenRevocationService, SecureJWTMiddleware, TokenManagementController };
```

**ENBD Banking Scenarios:**

```javascript
// 1. User logs out
POST /api/auth/logout
Headers: { Authorization: "Bearer <token>" }
// Response: Current session revoked

// 2. User logs out from all devices
POST /api/auth/logout-all
Headers: { Authorization: "Bearer <token>" }
// Response: All tokens revoked, must re-login everywhere

// 3. View active sessions
GET /api/auth/sessions
// Response:
{
  "sessions": [
    {
      "sessionId": "sess-123",
      "deviceInfo": { "device": "iPhone 13", "browser": "Safari" },
      "createdAt": "2024-11-10T10:00:00Z",
      "current": true
    },
    {
      "sessionId": "sess-456",
      "deviceInfo": { "device": "MacBook Pro", "browser": "Chrome" },
      "createdAt": "2024-11-09T15:30:00Z",
      "current": false
    }
  ]
}

// 4. Logout other devices
POST /api/auth/logout-others
// Response: All sessions except current revoked
```

**Best Practices:**

1. ✅ **Multiple revocation strategies** for flexibility
2. ✅ **TTL-based blacklist** to save memory
3. ✅ **Session management** for device tracking
4. ✅ **User-level revocation** for security incidents
5. ✅ **Audit logging** for compliance
6. ✅ **Redis for performance** with automatic cleanup
7. ✅ **Graceful handling** of revoked tokens

---

**Summary Q1-Q5:**
- JWT structure and internals ✅
- Secure token generation with multiple algorithms ✅
- Express middleware for authentication & authorization ✅
- Token refresh mechanisms with rotation ✅
- Comprehensive token revocation strategies ✅

**Next: Q6-Q10 will cover OAuth 2.0, token storage, security vulnerabilities, and advanced patterns.**
