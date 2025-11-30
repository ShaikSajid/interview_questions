# Q39: Authentication - JWT (JSON Web Tokens)

## 📋 Summary
JWT (JSON Web Token) is a compact, URL-safe token format for authentication and authorization. This guide covers **JWT structure**, signing algorithms, access/refresh token patterns, token validation, security best practices, and comprehensive banking examples for mobile app authentication.

---

## 🎯 What You'll Learn
- **JWT Structure**: Header, payload, signature
- **Signing Algorithms**: HS256 (HMAC), RS256 (RSA)
- **Access Tokens**: Short-lived authentication tokens
- **Refresh Tokens**: Long-lived tokens for refreshing access
- **Token Validation**: Signature verification, expiration checking
- **Security**: Token storage, XSS/CSRF protection, revocation
- **Banking Examples**: Secure mobile banking authentication system

---

## 📖 Comprehensive Explanation

### What is JWT?

**JWT** (JSON Web Token) is a standard (RFC 7519) for securely transmitting information between parties as a JSON object.

**Structure**: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### JWT Components

1. **Header** (Base64URL encoded)
   ```json
   {
     "alg": "HS256",    // Algorithm
     "typ": "JWT"       // Type
   }
   ```

2. **Payload** (Base64URL encoded)
   ```json
   {
     "sub": "user-123",        // Subject (user ID)
     "name": "John Doe",
     "iat": 1516239022,         // Issued at
     "exp": 1516242622          // Expiration
   }
   ```

3. **Signature**
   ```javascript
   HMACSHA256(
     base64UrlEncode(header) + "." + base64UrlEncode(payload),
     secret
   )
   ```

### Standard Claims

| Claim | Name | Description |
|-------|------|-------------|
| **iss** | Issuer | Who created the token |
| **sub** | Subject | User ID |
| **aud** | Audience | Intended recipient |
| **exp** | Expiration | Expiration timestamp |
| **nbf** | Not Before | Token not valid before this time |
| **iat** | Issued At | Token creation time |
| **jti** | JWT ID | Unique token identifier |

---

## 🔐 Access vs Refresh Tokens

### Access Token

**Purpose**: Short-lived token for API authentication

**Characteristics**:
- Short lifetime (5-15 minutes)
- Contains user identity and permissions
- Sent with every API request
- Cannot be revoked easily (stateless)

**Example**:
```json
{
  "sub": "user-123",
  "role": "customer",
  "permissions": ["view_balance", "transfer"],
  "iat": 1705315822,
  "exp": 1705316722  // 15 minutes
}
```

### Refresh Token

**Purpose**: Long-lived token to obtain new access tokens

**Characteristics**:
- Long lifetime (7-30 days)
- Stored securely (database)
- Used only to get new access tokens
- Can be revoked
- Rotated on use

**Example**:
```json
{
  "sub": "user-123",
  "type": "refresh",
  "jti": "refresh-abc123",  // Unique ID for revocation
  "iat": 1705315822,
  "exp": 1706525422  // 14 days
}
```

### Token Flow

```
1. Login → Server returns { accessToken, refreshToken }
2. API Request → Authorization: Bearer <accessToken>
3. Access token expires → Use refreshToken to get new accessToken
4. Logout → Revoke refreshToken
```

---

## 🔒 Signing Algorithms

### HS256 (HMAC + SHA256) - Symmetric

**How it works**: Single secret key for signing and verification

**Pros**:
- ✅ Simple
- ✅ Fast
- ✅ Less CPU intensive

**Cons**:
- ❌ Same key for signing and verification
- ❌ Every service needs the secret
- ❌ Key compromise = all tokens invalid

**Use case**: Single backend service

```javascript
const jwt = require('jsonwebtoken');

const secret = process.env.JWT_SECRET;
const token = jwt.sign({ userId: '123' }, secret, { algorithm: 'HS256' });
const decoded = jwt.verify(token, secret);
```

### RS256 (RSA + SHA256) - Asymmetric

**How it works**: Private key for signing, public key for verification

**Pros**:
- ✅ Public key can be shared
- ✅ Private key stays secure
- ✅ Better for microservices

**Cons**:
- ❌ More complex
- ❌ Slower (more CPU)
- ❌ Larger tokens

**Use case**: Microservices, public APIs

```javascript
const fs = require('fs');
const privateKey = fs.readFileSync('private.key');
const publicKey = fs.readFileSync('public.key');

// Sign with private key
const token = jwt.sign({ userId: '123' }, privateKey, { algorithm: 'RS256' });

// Verify with public key
const decoded = jwt.verify(token, publicKey);
```

---

## 📝 Example 1: Complete JWT Authentication System

### JWT Service

```javascript
// src/services/jwtService.js

const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const logger = require('../logger');

class JWTService {
  constructor() {
    this.accessTokenSecret = process.env.JWT_ACCESS_SECRET;
    this.refreshTokenSecret = process.env.JWT_REFRESH_SECRET;
    this.accessTokenExpiry = '15m';
    this.refreshTokenExpiry = '14d';
  }

  /**
   * Generate access token (short-lived)
   */
  generateAccessToken(user) {
    const payload = {
      sub: user.id,
      email: user.email,
      role: user.role,
      permissions: user.permissions || [],
      type: 'access'
    };

    return jwt.sign(payload, this.accessTokenSecret, {
      algorithm: 'HS256',
      expiresIn: this.accessTokenExpiry,
      issuer: 'banking-api',
      audience: 'banking-app'
    });
  }

  /**
   * Generate refresh token (long-lived)
   */
  generateRefreshToken(user) {
    const jti = crypto.randomBytes(16).toString('hex');

    const payload = {
      sub: user.id,
      type: 'refresh',
      jti  // Unique ID for revocation
    };

    const token = jwt.sign(payload, this.refreshTokenSecret, {
      algorithm: 'HS256',
      expiresIn: this.refreshTokenExpiry,
      issuer: 'banking-api'
    });

    return { token, jti };
  }

  /**
   * Verify access token
   */
  verifyAccessToken(token) {
    try {
      const decoded = jwt.verify(token, this.accessTokenSecret, {
        algorithms: ['HS256'],
        issuer: 'banking-api',
        audience: 'banking-app'
      });

      if (decoded.type !== 'access') {
        throw new Error('Invalid token type');
      }

      return decoded;

    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        throw new Error('Access token expired');
      }
      if (error.name === 'JsonWebTokenError') {
        throw new Error('Invalid access token');
      }
      throw error;
    }
  }

  /**
   * Verify refresh token
   */
  verifyRefreshToken(token) {
    try {
      const decoded = jwt.verify(token, this.refreshTokenSecret, {
        algorithms: ['HS256'],
        issuer: 'banking-api'
      });

      if (decoded.type !== 'refresh') {
        throw new Error('Invalid token type');
      }

      return decoded;

    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        throw new Error('Refresh token expired');
      }
      if (error.name === 'JsonWebTokenError') {
        throw new Error('Invalid refresh token');
      }
      throw error;
    }
  }

  /**
   * Decode token without verification (for inspection)
   */
  decodeToken(token) {
    return jwt.decode(token, { complete: true });
  }
}

module.exports = new JWTService();
```

### Refresh Token Repository

```javascript
// src/repositories/refreshTokenRepository.js

class RefreshTokenRepository {
  constructor(db) {
    this.db = db;
  }

  /**
   * Store refresh token in database
   */
  async storeRefreshToken(userId, jti, expiresAt) {
    const query = `
      INSERT INTO refresh_tokens (user_id, token_id, expires_at)
      VALUES ($1, $2, $3)
      RETURNING *
    `;

    const result = await this.db.query(query, [userId, jti, expiresAt]);
    return result.rows[0];
  }

  /**
   * Check if refresh token is valid (not revoked)
   */
  async isRefreshTokenValid(jti) {
    const query = `
      SELECT * FROM refresh_tokens
      WHERE token_id = $1
        AND revoked = false
        AND expires_at > NOW()
    `;

    const result = await this.db.query(query, [jti]);
    return result.rows.length > 0;
  }

  /**
   * Revoke refresh token
   */
  async revokeRefreshToken(jti) {
    const query = `
      UPDATE refresh_tokens
      SET revoked = true, revoked_at = NOW()
      WHERE token_id = $1
    `;

    await this.db.query(query, [jti]);
  }

  /**
   * Revoke all refresh tokens for user
   */
  async revokeAllUserTokens(userId) {
    const query = `
      UPDATE refresh_tokens
      SET revoked = true, revoked_at = NOW()
      WHERE user_id = $1 AND revoked = false
    `;

    await this.db.query(query, [userId]);
  }

  /**
   * Rotate refresh token (revoke old, create new)
   */
  async rotateRefreshToken(oldJti, userId, newJti, expiresAt) {
    const client = await this.db.connect();

    try {
      await client.query('BEGIN');

      // Revoke old token
      await client.query(
        'UPDATE refresh_tokens SET revoked = true WHERE token_id = $1',
        [oldJti]
      );

      // Store new token
      await client.query(
        'INSERT INTO refresh_tokens (user_id, token_id, expires_at) VALUES ($1, $2, $3)',
        [userId, newJti, expiresAt]
      );

      await client.query('COMMIT');

    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Cleanup expired tokens
   */
  async cleanupExpiredTokens() {
    const query = `
      DELETE FROM refresh_tokens
      WHERE expires_at < NOW() - INTERVAL '7 days'
    `;

    const result = await this.db.query(query);
    return result.rowCount;
  }
}

module.exports = RefreshTokenRepository;
```

### Authentication Controller

```javascript
// src/controllers/authController.js

const bcrypt = require('bcrypt');
const jwtService = require('../services/jwtService');
const RefreshTokenRepository = require('../repositories/refreshTokenRepository');
const logger = require('../logger');

class AuthController {
  constructor(db) {
    this.db = db;
    this.refreshTokenRepo = new RefreshTokenRepository(db);
  }

  /**
   * POST /api/auth/login
   * User login
   */
  async login(req, res) {
    try {
      const { email, password } = req.body;

      // Validate input
      if (!email || !password) {
        return res.status(400).json({
          error: 'Email and password are required'
        });
      }

      // Find user
      const result = await this.db.query(
        'SELECT * FROM users WHERE email = $1',
        [email]
      );

      if (result.rows.length === 0) {
        logger.warn('Login attempt for non-existent user', { email });
        return res.status(401).json({
          error: 'Invalid credentials'
        });
      }

      const user = result.rows[0];

      // Verify password
      const passwordValid = await bcrypt.compare(password, user.password_hash);
      if (!passwordValid) {
        logger.warn('Failed login attempt', { userId: user.id });
        return res.status(401).json({
          error: 'Invalid credentials'
        });
      }

      // Generate tokens
      const accessToken = jwtService.generateAccessToken({
        id: user.id,
        email: user.email,
        role: user.role,
        permissions: user.permissions
      });

      const { token: refreshToken, jti } = jwtService.generateRefreshToken({
        id: user.id
      });

      // Store refresh token
      const expiresAt = new Date(Date.now() + 14 * 24 * 60 * 60 * 1000); // 14 days
      await this.refreshTokenRepo.storeRefreshToken(user.id, jti, expiresAt);

      logger.info('User logged in', { userId: user.id });

      res.status(200).json({
        accessToken,
        refreshToken,
        expiresIn: 900,  // 15 minutes in seconds
        tokenType: 'Bearer',
        user: {
          id: user.id,
          email: user.email,
          role: user.role
        }
      });

    } catch (error) {
      logger.error('Login error', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * POST /api/auth/refresh
   * Refresh access token
   */
  async refresh(req, res) {
    try {
      const { refreshToken } = req.body;

      if (!refreshToken) {
        return res.status(400).json({
          error: 'Refresh token is required'
        });
      }

      // Verify refresh token
      let decoded;
      try {
        decoded = jwtService.verifyRefreshToken(refreshToken);
      } catch (error) {
        return res.status(401).json({
          error: error.message
        });
      }

      // Check if token is revoked
      const isValid = await this.refreshTokenRepo.isRefreshTokenValid(decoded.jti);
      if (!isValid) {
        logger.warn('Attempted use of revoked refresh token', { jti: decoded.jti });
        return res.status(401).json({
          error: 'Refresh token has been revoked'
        });
      }

      // Get user
      const result = await this.db.query(
        'SELECT * FROM users WHERE id = $1',
        [decoded.sub]
      );

      if (result.rows.length === 0) {
        return res.status(401).json({
          error: 'User not found'
        });
      }

      const user = result.rows[0];

      // Generate new access token
      const accessToken = jwtService.generateAccessToken({
        id: user.id,
        email: user.email,
        role: user.role,
        permissions: user.permissions
      });

      // Rotate refresh token (optional but recommended)
      const { token: newRefreshToken, jti: newJti } = jwtService.generateRefreshToken({
        id: user.id
      });

      const expiresAt = new Date(Date.now() + 14 * 24 * 60 * 60 * 1000);
      await this.refreshTokenRepo.rotateRefreshToken(
        decoded.jti,
        user.id,
        newJti,
        expiresAt
      );

      logger.info('Access token refreshed', { userId: user.id });

      res.status(200).json({
        accessToken,
        refreshToken: newRefreshToken,
        expiresIn: 900,
        tokenType: 'Bearer'
      });

    } catch (error) {
      logger.error('Token refresh error', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * POST /api/auth/logout
   * User logout (revoke refresh token)
   */
  async logout(req, res) {
    try {
      const { refreshToken } = req.body;

      if (!refreshToken) {
        return res.status(400).json({
          error: 'Refresh token is required'
        });
      }

      // Decode without verification (we just need the jti)
      const decoded = jwtService.decodeToken(refreshToken);
      if (decoded && decoded.payload.jti) {
        await this.refreshTokenRepo.revokeRefreshToken(decoded.payload.jti);
        logger.info('User logged out', { userId: decoded.payload.sub });
      }

      res.status(200).json({
        message: 'Logged out successfully'
      });

    } catch (error) {
      logger.error('Logout error', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }

  /**
   * POST /api/auth/logout-all
   * Logout from all devices
   */
  async logoutAll(req, res) {
    try {
      const userId = req.user.id;  // From auth middleware

      await this.refreshTokenRepo.revokeAllUserTokens(userId);

      logger.info('User logged out from all devices', { userId });

      res.status(200).json({
        message: 'Logged out from all devices'
      });

    } catch (error) {
      logger.error('Logout all error', { error: error.message });
      res.status(500).json({ error: 'Internal server error' });
    }
  }
}

module.exports = AuthController;
```

### Authentication Middleware

```javascript
// src/middleware/authMiddleware.js

const jwtService = require('../services/jwtService');
const logger = require('../logger');

function authMiddleware(req, res, next) {
  try {
    // Get token from header
    const authHeader = req.headers.authorization;

    if (!authHeader) {
      return res.status(401).json({
        error: 'Authorization header required'
      });
    }

    // Extract token (format: "Bearer <token>")
    const parts = authHeader.split(' ');
    if (parts.length !== 2 || parts[0] !== 'Bearer') {
      return res.status(401).json({
        error: 'Invalid authorization format. Use: Bearer <token>'
      });
    }

    const token = parts[1];

    // Verify token
    try {
      const decoded = jwtService.verifyAccessToken(token);
      
      // Attach user to request
      req.user = {
        id: decoded.sub,
        email: decoded.email,
        role: decoded.role,
        permissions: decoded.permissions
      };

      next();

    } catch (error) {
      logger.warn('Invalid access token', { error: error.message });
      return res.status(401).json({
        error: error.message
      });
    }

  } catch (error) {
    logger.error('Auth middleware error', { error: error.message });
    res.status(500).json({ error: 'Internal server error' });
  }
}

/**
 * Role-based authorization middleware
 */
function requireRole(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({
        error: 'Authentication required'
      });
    }

    if (!allowedRoles.includes(req.user.role)) {
      logger.warn('Unauthorized access attempt', {
        userId: req.user.id,
        role: req.user.role,
        requiredRoles: allowedRoles
      });
      
      return res.status(403).json({
        error: 'Insufficient permissions',
        requiredRoles: allowedRoles
      });
    }

    next();
  };
}

/**
 * Permission-based authorization middleware
 */
function requirePermission(...requiredPermissions) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({
        error: 'Authentication required'
      });
    }

    const userPermissions = req.user.permissions || [];
    const hasPermission = requiredPermissions.every(p => userPermissions.includes(p));

    if (!hasPermission) {
      logger.warn('Unauthorized access attempt', {
        userId: req.user.id,
        userPermissions,
        requiredPermissions
      });
      
      return res.status(403).json({
        error: 'Insufficient permissions',
        requiredPermissions
      });
    }

    next();
  };
}

module.exports = {
  authMiddleware,
  requireRole,
  requirePermission
};
```

### Protected Routes Example

```javascript
// src/routes/accounts.js

const express = require('express');
const { authMiddleware, requireRole, requirePermission } = require('../middleware/authMiddleware');

const router = express.Router();

// Public route (no auth)
router.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Protected route (requires authentication)
router.get('/accounts', authMiddleware, async (req, res) => {
  // req.user available here
  res.json({ accounts: [] });
});

// Admin-only route
router.get('/admin/users', authMiddleware, requireRole('admin'), async (req, res) => {
  res.json({ users: [] });
});

// Permission-based route
router.post('/transfer', authMiddleware, requirePermission('transfer'), async (req, res) => {
  res.json({ status: 'success' });
});

module.exports = router;
```

### Database Schema

```sql
-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(50) DEFAULT 'customer',
  permissions JSONB DEFAULT '[]',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Refresh tokens table
CREATE TABLE refresh_tokens (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  token_id VARCHAR(255) UNIQUE NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  revoked BOOLEAN DEFAULT false,
  revoked_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token_id ON refresh_tokens(token_id);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens(expires_at);
```

---

## 🔒 Security Best Practices

### 1. Token Storage

**❌ Don't store in localStorage** (vulnerable to XSS)
```javascript
localStorage.setItem('accessToken', token);  // BAD
```

**✅ Store in httpOnly cookie** (protected from XSS)
```javascript
res.cookie('accessToken', token, {
  httpOnly: true,
  secure: true,      // HTTPS only
  sameSite: 'strict',
  maxAge: 900000     // 15 minutes
});
```

### 2. Token Expiration

- **Access tokens**: 5-15 minutes
- **Refresh tokens**: 7-30 days
- **Shorter = more secure**

### 3. Refresh Token Rotation

Rotate refresh tokens on every use:
```javascript
// Old refresh token → New access token + New refresh token
// Old refresh token is revoked
```

### 4. Token Revocation

Store refresh tokens in database for revocation:
- Logout
- Password change
- Suspicious activity
- Admin action

### 5. HTTPS Only

Always use HTTPS in production:
```javascript
if (process.env.NODE_ENV === 'production' && !req.secure) {
  return res.redirect('https://' + req.headers.host + req.url);
}
```

---

## 🎯 Key Takeaways

### JWT Best Practices

1. **Short access token lifetime** (5-15 minutes)
2. **Use refresh tokens** for long sessions
3. **Rotate refresh tokens** on use
4. **Store refresh tokens in database** (for revocation)
5. **Use HS256 for single service**, RS256 for microservices
6. **Never store secrets in JWT** (tokens can be decoded)
7. **Validate exp, iss, aud** claims
8. **Use HTTPS** in production

### Common Mistakes

- ❌ Storing sensitive data in JWT payload
- ❌ Long-lived access tokens (hours/days)
- ❌ No refresh token rotation
- ❌ Storing tokens in localStorage
- ❌ Not validating token claims
- ❌ Using weak secrets (< 256 bits)

---

**File**: `Q39_Authentication_JWT.md`  
**Status**: ✅ Complete with JWT structure, access/refresh tokens, middleware, security practices, and complete banking authentication system
