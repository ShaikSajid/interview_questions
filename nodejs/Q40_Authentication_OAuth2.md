# Q40: Authentication - OAuth 2.0

## 📋 Summary
OAuth 2.0 is an authorization framework that enables third-party applications to obtain limited access to user accounts. This guide covers **OAuth 2.0 flows**, grant types, scopes, PKCE, OpenID Connect, and comprehensive banking examples for "Login with Bank" integration.

---

## 🎯 What You'll Learn
- **OAuth 2.0 Basics**: Authorization vs authentication, roles, flows
- **Grant Types**: Authorization Code, Client Credentials, Refresh Token
- **PKCE**: Proof Key for Code Exchange (for mobile/SPA)
- **Scopes & Permissions**: Granular access control
- **OpenID Connect**: Authentication layer on top of OAuth 2.0
- **Banking Examples**: Open Banking API, fintech app integration

---

## 📖 Comprehensive Explanation

### What is OAuth 2.0?

**OAuth 2.0** is an **authorization** framework (not authentication) that allows third-party apps to access user resources without exposing credentials.

**Key Concept**: **Delegated Authorization**

> "Allow App X to access my banking data without giving App X my banking password."

### OAuth vs Authentication

| OAuth 2.0 | Authentication (e.g., JWT) |
|-----------|----------------------------|
| Authorization (what you can do) | Authentication (who you are) |
| Delegated access | Direct access |
| Third-party apps | First-party apps |
| Example: "Login with Google" | Example: Username/password |

---

## 👥 OAuth 2.0 Roles

1. **Resource Owner** (User)
   - Person who owns the data
   - Example: Bank customer

2. **Client** (Third-party App)
   - Application requesting access
   - Example: Budgeting app

3. **Authorization Server**
   - Issues access tokens
   - Example: Bank's OAuth server

4. **Resource Server** (API)
   - Hosts protected resources
   - Example: Bank's account API

---

## 🔄 OAuth 2.0 Flows

### 1. Authorization Code Flow (Most Common)

**Use case**: Web applications with backend

**Flow**:
```
1. User → Client: "I want to use this app"
2. Client → Authorization Server: "Redirect user to login"
3. User → Authorization Server: "Login and consent"
4. Authorization Server → Client: "Here's an authorization code"
5. Client → Authorization Server: "Exchange code for access token" (with client secret)
6. Authorization Server → Client: "Here's an access token"
7. Client → Resource Server: "Access user data" (with access token)
```

**Diagram**:
```
User -> App -> Bank Login Page -> User approves ->
Bank sends code -> App exchanges code for token -> App accesses data
```

### 2. Authorization Code Flow + PKCE

**Use case**: Mobile apps, SPAs (no client secret)

**PKCE** (Proof Key for Code Exchange) adds security:
- App generates `code_verifier` (random string)
- App sends `code_challenge` = SHA256(code_verifier)
- When exchanging code, app sends `code_verifier`
- Server verifies: SHA256(code_verifier) == code_challenge

**Why**: Prevents authorization code interception attacks

### 3. Client Credentials Flow

**Use case**: Machine-to-machine (no user involved)

**Flow**:
```
App -> Authorization Server: "Here's my client_id and client_secret"
Authorization Server -> App: "Here's an access token"
App -> API: "Access resources" (with token)
```

**Example**: Cron job that syncs bank data

### 4. Refresh Token Flow

**Use case**: Get new access token without user re-authentication

**Flow**:
```
App -> Authorization Server: "Here's my refresh token"
Authorization Server -> App: "Here's a new access token"
```

---

## 🔑 Scopes and Permissions

**Scopes** define what access the client is requesting.

**Example Banking Scopes**:
```
accounts:read          # Read account details
accounts:balance:read  # Read account balance
transactions:read      # Read transaction history
transfers:create       # Initiate transfers (requires consent)
```

**Authorization Request**:
```
GET /oauth/authorize?
  response_type=code&
  client_id=app123&
  redirect_uri=https://app.com/callback&
  scope=accounts:read transactions:read&
  state=xyz
```

---

## 📝 Example 1: OAuth 2.0 Authorization Server (Bank)

### Authorization Server Setup

```javascript
// src/oauth/authorizationServer.js

const express = require('express');
const crypto = require('crypto');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

class AuthorizationServer {
  constructor(db) {
    this.db = db;
    this.authorizationCodes = new Map(); // In-memory for demo (use Redis in production)
  }

  /**
   * GET /oauth/authorize
   * Authorization endpoint (user consent)
   */
  async authorize(req, res) {
    try {
      const {
        response_type,
        client_id,
        redirect_uri,
        scope,
        state,
        code_challenge,        // PKCE
        code_challenge_method  // PKCE (S256)
      } = req.query;

      // Validate parameters
      if (response_type !== 'code') {
        return res.status(400).json({
          error: 'unsupported_response_type',
          error_description: 'Only authorization code flow is supported'
        });
      }

      // Validate client
      const client = await this.getClient(client_id);
      if (!client) {
        return res.status(400).json({
          error: 'invalid_client',
          error_description: 'Client not found'
        });
      }

      // Validate redirect URI
      if (!client.redirect_uris.includes(redirect_uri)) {
        return res.status(400).json({
          error: 'invalid_redirect_uri',
          error_description: 'Redirect URI not registered'
        });
      }

      // Validate scopes
      const requestedScopes = scope.split(' ');
      const validScopes = ['accounts:read', 'transactions:read', 'transfers:create'];
      const invalidScopes = requestedScopes.filter(s => !validScopes.includes(s));
      
      if (invalidScopes.length > 0) {
        return res.status(400).json({
          error: 'invalid_scope',
          error_description: `Invalid scopes: ${invalidScopes.join(', ')}`
        });
      }

      // Check if user is logged in
      if (!req.session.userId) {
        // Redirect to login page
        req.session.oauth = {
          client_id,
          redirect_uri,
          scope,
          state,
          code_challenge,
          code_challenge_method
        };
        return res.redirect('/login?returnTo=/oauth/authorize');
      }

      // Show consent screen
      res.render('consent', {
        client: client.name,
        scopes: requestedScopes,
        approveUrl: '/oauth/approve',
        denyUrl: '/oauth/deny'
      });

    } catch (error) {
      console.error('Authorization error:', error);
      res.status(500).json({ error: 'server_error' });
    }
  }

  /**
   * POST /oauth/approve
   * User approves consent
   */
  async approve(req, res) {
    try {
      const userId = req.session.userId;
      const oauthParams = req.session.oauth;

      if (!userId || !oauthParams) {
        return res.status(400).json({ error: 'invalid_request' });
      }

      // Generate authorization code
      const code = crypto.randomBytes(32).toString('hex');

      // Store authorization code
      this.authorizationCodes.set(code, {
        userId,
        clientId: oauthParams.client_id,
        redirectUri: oauthParams.redirect_uri,
        scope: oauthParams.scope,
        codeChallenge: oauthParams.code_challenge,
        codeChallengeMethod: oauthParams.code_challenge_method,
        expiresAt: Date.now() + 10 * 60 * 1000  // 10 minutes
      });

      // Clear session
      delete req.session.oauth;

      // Redirect back to client with authorization code
      const redirectUrl = new URL(oauthParams.redirect_uri);
      redirectUrl.searchParams.set('code', code);
      if (oauthParams.state) {
        redirectUrl.searchParams.set('state', oauthParams.state);
      }

      res.redirect(redirectUrl.toString());

    } catch (error) {
      console.error('Approval error:', error);
      res.status(500).json({ error: 'server_error' });
    }
  }

  /**
   * POST /oauth/token
   * Token endpoint (exchange code for access token)
   */
  async token(req, res) {
    try {
      const {
        grant_type,
        code,
        redirect_uri,
        client_id,
        client_secret,
        code_verifier,     // PKCE
        refresh_token
      } = req.body;

      // Authorization Code Grant
      if (grant_type === 'authorization_code') {
        return await this.handleAuthorizationCodeGrant(
          code,
          redirect_uri,
          client_id,
          client_secret,
          code_verifier,
          res
        );
      }

      // Refresh Token Grant
      if (grant_type === 'refresh_token') {
        return await this.handleRefreshTokenGrant(
          refresh_token,
          client_id,
          client_secret,
          res
        );
      }

      // Client Credentials Grant
      if (grant_type === 'client_credentials') {
        return await this.handleClientCredentialsGrant(
          client_id,
          client_secret,
          res
        );
      }

      res.status(400).json({
        error: 'unsupported_grant_type'
      });

    } catch (error) {
      console.error('Token error:', error);
      res.status(500).json({ error: 'server_error' });
    }
  }

  /**
   * Handle authorization code grant
   */
  async handleAuthorizationCodeGrant(
    code,
    redirect_uri,
    client_id,
    client_secret,
    code_verifier,
    res
  ) {
    // Validate authorization code
    const authData = this.authorizationCodes.get(code);
    if (!authData) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Invalid authorization code'
      });
    }

    // Check expiration
    if (Date.now() > authData.expiresAt) {
      this.authorizationCodes.delete(code);
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Authorization code expired'
      });
    }

    // Validate client
    const client = await this.getClient(client_id);
    if (!client) {
      return res.status(400).json({ error: 'invalid_client' });
    }

    // Validate client secret (if confidential client)
    if (client.client_type === 'confidential') {
      const secretValid = await bcrypt.compare(client_secret, client.client_secret_hash);
      if (!secretValid) {
        return res.status(401).json({ error: 'invalid_client' });
      }
    }

    // Validate PKCE (if used)
    if (authData.codeChallenge) {
      if (!code_verifier) {
        return res.status(400).json({
          error: 'invalid_request',
          error_description: 'code_verifier required'
        });
      }

      const challenge = crypto
        .createHash('sha256')
        .update(code_verifier)
        .digest('base64url');

      if (challenge !== authData.codeChallenge) {
        return res.status(400).json({
          error: 'invalid_grant',
          error_description: 'Invalid code_verifier'
        });
      }
    }

    // Validate redirect URI
    if (redirect_uri !== authData.redirectUri) {
      return res.status(400).json({ error: 'invalid_grant' });
    }

    // Delete authorization code (single use)
    this.authorizationCodes.delete(code);

    // Generate tokens
    const accessToken = this.generateAccessToken({
      userId: authData.userId,
      clientId: client_id,
      scope: authData.scope
    });

    const refreshToken = this.generateRefreshToken({
      userId: authData.userId,
      clientId: client_id,
      scope: authData.scope
    });

    // Store refresh token
    await this.storeRefreshToken(refreshToken, authData.userId, client_id);

    res.status(200).json({
      access_token: accessToken,
      token_type: 'Bearer',
      expires_in: 3600,  // 1 hour
      refresh_token: refreshToken,
      scope: authData.scope
    });
  }

  /**
   * Handle refresh token grant
   */
  async handleRefreshTokenGrant(refresh_token, client_id, client_secret, res) {
    // Validate refresh token
    const tokenData = await this.validateRefreshToken(refresh_token);
    if (!tokenData) {
      return res.status(400).json({
        error: 'invalid_grant',
        error_description: 'Invalid refresh token'
      });
    }

    // Validate client
    if (tokenData.clientId !== client_id) {
      return res.status(400).json({ error: 'invalid_client' });
    }

    // Generate new access token
    const accessToken = this.generateAccessToken({
      userId: tokenData.userId,
      clientId: client_id,
      scope: tokenData.scope
    });

    res.status(200).json({
      access_token: accessToken,
      token_type: 'Bearer',
      expires_in: 3600,
      scope: tokenData.scope
    });
  }

  /**
   * Handle client credentials grant (machine-to-machine)
   */
  async handleClientCredentialsGrant(client_id, client_secret, res) {
    // Validate client
    const client = await this.getClient(client_id);
    if (!client) {
      return res.status(400).json({ error: 'invalid_client' });
    }

    const secretValid = await bcrypt.compare(client_secret, client.client_secret_hash);
    if (!secretValid) {
      return res.status(401).json({ error: 'invalid_client' });
    }

    // Generate access token
    const accessToken = this.generateAccessToken({
      clientId: client_id,
      scope: client.default_scope
    });

    res.status(200).json({
      access_token: accessToken,
      token_type: 'Bearer',
      expires_in: 3600,
      scope: client.default_scope
    });
  }

  generateAccessToken(payload) {
    return jwt.sign(payload, process.env.OAUTH_ACCESS_SECRET, {
      algorithm: 'HS256',
      expiresIn: '1h'
    });
  }

  generateRefreshToken(payload) {
    return jwt.sign(
      { ...payload, type: 'refresh' },
      process.env.OAUTH_REFRESH_SECRET,
      { algorithm: 'HS256', expiresIn: '30d' }
    );
  }

  async getClient(client_id) {
    const result = await this.db.query(
      'SELECT * FROM oauth_clients WHERE client_id = $1',
      [client_id]
    );
    return result.rows[0];
  }

  async storeRefreshToken(token, userId, clientId) {
    await this.db.query(
      'INSERT INTO oauth_refresh_tokens (token, user_id, client_id) VALUES ($1, $2, $3)',
      [token, userId, clientId]
    );
  }

  async validateRefreshToken(token) {
    try {
      const decoded = jwt.verify(token, process.env.OAUTH_REFRESH_SECRET);
      
      // Check if token is revoked
      const result = await this.db.query(
        'SELECT * FROM oauth_refresh_tokens WHERE token = $1 AND revoked = false',
        [token]
      );

      if (result.rows.length === 0) {
        return null;
      }

      return {
        userId: decoded.userId,
        clientId: decoded.clientId,
        scope: decoded.scope
      };
    } catch (error) {
      return null;
    }
  }
}

module.exports = AuthorizationServer;
```

### Resource Server (Protected API)

```javascript
// src/middleware/oauthMiddleware.js

const jwt = require('jsonwebtoken');

function oauthMiddleware(requiredScopes = []) {
  return (req, res, next) => {
    try {
      // Extract token
      const authHeader = req.headers.authorization;
      if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return res.status(401).json({
          error: 'invalid_token',
          error_description: 'Missing or invalid authorization header'
        });
      }

      const token = authHeader.substring(7);

      // Verify token
      const decoded = jwt.verify(token, process.env.OAUTH_ACCESS_SECRET);

      // Check scopes
      const tokenScopes = decoded.scope ? decoded.scope.split(' ') : [];
      const hasRequiredScopes = requiredScopes.every(s => tokenScopes.includes(s));

      if (!hasRequiredScopes) {
        return res.status(403).json({
          error: 'insufficient_scope',
          error_description: `Required scopes: ${requiredScopes.join(', ')}`,
          scopes: requiredScopes
        });
      }

      // Attach token data to request
      req.oauth = {
        userId: decoded.userId,
        clientId: decoded.clientId,
        scopes: tokenScopes
      };

      next();

    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        return res.status(401).json({
          error: 'invalid_token',
          error_description: 'Access token expired'
        });
      }
      
      return res.status(401).json({
        error: 'invalid_token',
        error_description: 'Invalid access token'
      });
    }
  };
}

module.exports = oauthMiddleware;
```

### Protected Banking API Routes

```javascript
// src/routes/openBanking.js

const express = require('express');
const oauthMiddleware = require('../middleware/oauthMiddleware');

const router = express.Router();

// Get accounts (requires accounts:read scope)
router.get('/accounts', oauthMiddleware(['accounts:read']), async (req, res) => {
  const userId = req.oauth.userId;
  
  // Fetch user accounts
  const accounts = await getAccountsForUser(userId);
  
  res.json({
    data: accounts,
    _links: {
      self: '/api/open-banking/accounts'
    }
  });
});

// Get transactions (requires transactions:read scope)
router.get('/accounts/:accountId/transactions', 
  oauthMiddleware(['transactions:read']), 
  async (req, res) => {
    const { accountId } = req.params;
    const userId = req.oauth.userId;
    
    // Verify account belongs to user
    const accountBelongsToUser = await verifyAccountOwnership(accountId, userId);
    if (!accountBelongsToUser) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    const transactions = await getTransactionsForAccount(accountId);
    
    res.json({
      data: transactions
    });
  }
);

// Create transfer (requires transfers:create scope)
router.post('/transfers', 
  oauthMiddleware(['transfers:create']), 
  async (req, res) => {
    const { fromAccount, toAccount, amount } = req.body;
    const userId = req.oauth.userId;
    
    // Create transfer
    const transfer = await createTransfer(userId, fromAccount, toAccount, amount);
    
    res.status(201).json({
      data: transfer
    });
  }
);

module.exports = router;
```

---

## 🎯 Key Takeaways

### OAuth 2.0 Best Practices

1. **Use Authorization Code + PKCE** for mobile/SPA
2. **Always use state parameter** (CSRF protection)
3. **Short-lived access tokens** (1 hour)
4. **Refresh token rotation** (issue new refresh token on use)
5. **Validate redirect URIs** (exact match, no wildcards)
6. **Use scopes** for granular permissions
7. **HTTPS only** in production

### Common Mistakes

- ❌ Using Implicit Flow (deprecated, insecure)
- ❌ Long-lived access tokens
- ❌ Not validating redirect URIs
- ❌ Storing tokens in localStorage
- ❌ Not using PKCE for mobile/SPA
- ❌ Not implementing token revocation

### OAuth 2.0 vs OpenID Connect

**OAuth 2.0**: Authorization ("What can I do?")
**OpenID Connect**: Authentication + Authorization ("Who am I?")

OpenID Connect adds:
- `id_token` (JWT with user info)
- `/userinfo` endpoint
- Standard claims (name, email, picture)

---

**File**: `Q40_Authentication_OAuth2.md`  
**Status**: ✅ Complete with OAuth 2.0 flows, grant types, PKCE, scopes, and complete Open Banking integration example
