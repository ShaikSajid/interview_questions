# Q27: Security - Authentication

## 📋 Summary
This question covers **authentication and authorization** in Node.js applications, including JWT (JSON Web Tokens), session-based authentication, OAuth 2.0 flows, and secure password hashing with bcrypt. You'll learn how to implement production-grade authentication systems for banking applications handling sensitive customer data.

**What You'll Learn**:
- JWT authentication with access and refresh tokens
- Session-based authentication with secure cookies
- OAuth 2.0 authorization flows
- Password hashing with bcrypt and security best practices
- Multi-factor authentication (MFA) implementation
- Role-based access control (RBAC)
- Token storage and security considerations
- Authentication middleware patterns

---

## 🎯 Comprehensive Explanation

### Authentication vs Authorization

**Authentication** (Who are you?):
- Verifies user identity
- Credentials: username/password, tokens, biometrics
- Answers: "Is this user who they claim to be?"

**Authorization** (What can you do?):
- Determines permissions and access rights
- Role-based, resource-based, attribute-based
- Answers: "Is this user allowed to perform this action?"

### Authentication Strategies

#### 1. **Session-Based Authentication**

Traditional approach using server-side sessions:

```
Client                          Server
  |                               |
  |-- Login (credentials) ------> |
  |                               | Verify credentials
  |                               | Create session
  |                               | Store session in memory/DB
  | <----- Session Cookie ------- |
  |                               |
  |-- Request + Cookie ---------> |
  |                               | Lookup session
  |                               | Verify session
  | <----- Response ------------- |
```

**Pros**:
- Server controls all sessions
- Easy to revoke (delete session)
- Smaller payload in cookies

**Cons**:
- Requires server-side storage
- Doesn't scale well horizontally
- CSRF vulnerability if not protected

#### 2. **Token-Based Authentication (JWT)**

Stateless approach using signed tokens:

```
Client                          Server
  |                               |
  |-- Login (credentials) ------> |
  |                               | Verify credentials
  |                               | Generate JWT token
  | <----- JWT Token ------------ |
  |                               |
  |-- Request + JWT Header -----> |
  |                               | Verify JWT signature
  |                               | Decode payload
  | <----- Response ------------- |
```

**JWT Structure**:
```
Header.Payload.Signature

Header: { "alg": "HS256", "typ": "JWT" }
Payload: { "userId": 123, "role": "customer", "exp": 1700000000 }
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

**Pros**:
- Stateless (no server storage)
- Scales horizontally
- Can be used across domains

**Cons**:
- Cannot revoke easily (must wait for expiration)
- Larger payload than sessions
- Token theft risks if not stored securely

#### 3. **OAuth 2.0**

Delegated authorization protocol:

```
User → Your App → Identity Provider (Google, GitHub) → Your App
```

**OAuth Flows**:
1. **Authorization Code Flow**: Most secure for web apps
2. **Implicit Flow**: Deprecated, insecure
3. **Client Credentials Flow**: Machine-to-machine
4. **Password Flow**: Direct credentials (avoid if possible)

### Password Security with bcrypt

**Why bcrypt?**
- Adaptive hashing (configurable work factor)
- Built-in salt generation
- Resistant to rainbow table attacks
- Time-tested and secure

**Cost Factor**:
```javascript
// Cost factor of 10 = 2^10 = 1024 iterations (~65ms)
// Cost factor of 12 = 2^12 = 4096 iterations (~260ms)
// Cost factor of 14 = 2^14 = 16384 iterations (~1s)
```

**Never**:
- Store plain text passwords
- Use simple hashing (MD5, SHA1)
- Use encryption (reversible)

**Always**:
- Use bcrypt/argon2/scrypt
- Use cost factor ≥ 10
- Hash on server-side only

### Security Best Practices

1. **HTTPS Only**: All auth traffic over TLS
2. **Secure Cookie Flags**: httpOnly, secure, sameSite
3. **Token Expiration**: Short-lived access tokens (15 min)
4. **Refresh Tokens**: Long-lived but revocable
5. **Rate Limiting**: Prevent brute force attacks
6. **Account Lockout**: After N failed attempts
7. **MFA**: Two-factor authentication for sensitive operations
8. **CSRF Protection**: For session-based auth
9. **XSS Prevention**: Sanitize inputs, CSP headers
10. **Password Policies**: Minimum length, complexity, no common passwords

---

## 🏦 Real-World Banking Scenario

**Challenge**: Build a secure authentication system for an online banking platform that handles:
- 500,000+ active customers
- Multi-device access (web, mobile, tablet)
- High-value transactions requiring MFA
- Role-based access (customer, teller, manager, admin)
- Compliance with PCI DSS and banking regulations
- Session management across multiple concurrent logins
- Secure password reset and account recovery

**Requirements**:
1. JWT-based authentication with refresh tokens
2. bcrypt password hashing with cost factor 12
3. Multi-factor authentication for transactions >$10,000
4. Role-based access control with permissions
5. Audit logging for all authentication events
6. Rate limiting to prevent brute force attacks
7. Automatic token refresh without re-login
8. Secure logout and session invalidation

---

## 💻 Example 1: JWT Authentication with Refresh Tokens

This example implements a complete JWT authentication system with access and refresh tokens for a banking API.

```javascript
// auth-service.js
const express = require('express');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const app = express();

app.use(express.json());

// Configuration
const CONFIG = {
  JWT_ACCESS_SECRET: process.env.JWT_ACCESS_SECRET || crypto.randomBytes(64).toString('hex'),
  JWT_REFRESH_SECRET: process.env.JWT_REFRESH_SECRET || crypto.randomBytes(64).toString('hex'),
  ACCESS_TOKEN_EXPIRY: '15m',   // 15 minutes
  REFRESH_TOKEN_EXPIRY: '7d',   // 7 days
  BCRYPT_ROUNDS: 12,            // 2^12 iterations
  MAX_LOGIN_ATTEMPTS: 5,
  LOCKOUT_DURATION: 15 * 60 * 1000 // 15 minutes
};

// Simulated database
const users = new Map();
const refreshTokens = new Map(); // In production, use Redis
const loginAttempts = new Map();

// User model
class User {
  constructor(id, email, passwordHash, role, accountNumber) {
    this.id = id;
    this.email = email;
    this.passwordHash = passwordHash;
    this.role = role; // 'customer', 'teller', 'manager', 'admin'
    this.accountNumber = accountNumber;
    this.mfaEnabled = false;
    this.mfaSecret = null;
    this.createdAt = new Date();
    this.lastLogin = null;
  }
}

// Initialize sample users
async function initializeUsers() {
  const users = [
    { email: 'customer@bank.com', password: 'Customer123!', role: 'customer', accountNumber: 'ACC001' },
    { email: 'teller@bank.com', password: 'Teller123!', role: 'teller', accountNumber: 'EMP001' },
    { email: 'manager@bank.com', password: 'Manager123!', role: 'manager', accountNumber: 'MGR001' }
  ];

  for (const userData of users) {
    const passwordHash = await bcrypt.hash(userData.password, CONFIG.BCRYPT_ROUNDS);
    const user = new User(
      crypto.randomUUID(),
      userData.email,
      passwordHash,
      userData.role,
      userData.accountNumber
    );
    users.set(user.email, user);
  }

  console.log('Sample users initialized:');
  console.log('- customer@bank.com / Customer123!');
  console.log('- teller@bank.com / Teller123!');
  console.log('- manager@bank.com / Manager123!');
}

// Helper: Check if account is locked
function isAccountLocked(email) {
  const attempts = loginAttempts.get(email);
  if (!attempts) return false;

  if (attempts.count >= CONFIG.MAX_LOGIN_ATTEMPTS) {
    const lockoutEnd = attempts.lockedUntil;
    if (lockoutEnd && Date.now() < lockoutEnd) {
      return true;
    } else {
      // Lockout expired, reset attempts
      loginAttempts.delete(email);
      return false;
    }
  }

  return false;
}

// Helper: Record failed login attempt
function recordFailedAttempt(email) {
  const attempts = loginAttempts.get(email) || { count: 0, lockedUntil: null };
  attempts.count++;

  if (attempts.count >= CONFIG.MAX_LOGIN_ATTEMPTS) {
    attempts.lockedUntil = Date.now() + CONFIG.LOCKOUT_DURATION;
    console.log(`Account ${email} locked until ${new Date(attempts.lockedUntil).toISOString()}`);
  }

  loginAttempts.set(email, attempts);
}

// Helper: Reset login attempts
function resetLoginAttempts(email) {
  loginAttempts.delete(email);
}

// Generate access token
function generateAccessToken(user) {
  const payload = {
    userId: user.id,
    email: user.email,
    role: user.role,
    accountNumber: user.accountNumber,
    type: 'access'
  };

  return jwt.sign(payload, CONFIG.JWT_ACCESS_SECRET, {
    expiresIn: CONFIG.ACCESS_TOKEN_EXPIRY,
    issuer: 'banking-api',
    audience: 'banking-clients'
  });
}

// Generate refresh token
function generateRefreshToken(user) {
  const payload = {
    userId: user.id,
    email: user.email,
    type: 'refresh',
    tokenId: crypto.randomUUID() // Unique token ID for revocation
  };

  const token = jwt.sign(payload, CONFIG.JWT_REFRESH_SECRET, {
    expiresIn: CONFIG.REFRESH_TOKEN_EXPIRY,
    issuer: 'banking-api',
    audience: 'banking-clients'
  });

  // Store refresh token for validation
  refreshTokens.set(payload.tokenId, {
    userId: user.id,
    token,
    createdAt: new Date(),
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  return { token, tokenId: payload.tokenId };
}

// Middleware: Verify access token
function authenticateToken(req, res, next) {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, CONFIG.JWT_ACCESS_SECRET, (err, decoded) => {
    if (err) {
      if (err.name === 'TokenExpiredError') {
        return res.status(401).json({ error: 'Access token expired', code: 'TOKEN_EXPIRED' });
      }
      return res.status(403).json({ error: 'Invalid access token' });
    }

    if (decoded.type !== 'access') {
      return res.status(403).json({ error: 'Invalid token type' });
    }

    req.user = decoded;
    next();
  });
}

// Middleware: Check role authorization
function authorizeRole(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
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

// Route: Register new user
app.post('/auth/register', async (req, res) => {
  try {
    const { email, password, accountNumber } = req.body;

    // Validation
    if (!email || !password || !accountNumber) {
      return res.status(400).json({ error: 'Email, password, and account number required' });
    }

    if (users.has(email)) {
      return res.status(409).json({ error: 'User already exists' });
    }

    // Password validation
    if (password.length < 8) {
      return res.status(400).json({ error: 'Password must be at least 8 characters' });
    }

    // Hash password
    console.time(`Password hashing for ${email}`);
    const passwordHash = await bcrypt.hash(password, CONFIG.BCRYPT_ROUNDS);
    console.timeEnd(`Password hashing for ${email}`);

    // Create user
    const user = new User(
      crypto.randomUUID(),
      email,
      passwordHash,
      'customer', // Default role
      accountNumber
    );

    users.set(email, user);

    res.status(201).json({
      message: 'User registered successfully',
      userId: user.id,
      email: user.email,
      role: user.role
    });

  } catch (error) {
    console.error('Registration error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Route: Login
app.post('/auth/login', async (req, res) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password required' });
    }

    // Check if account is locked
    if (isAccountLocked(email)) {
      const attempts = loginAttempts.get(email);
      const remainingTime = Math.ceil((attempts.lockedUntil - Date.now()) / 1000 / 60);
      return res.status(429).json({ 
        error: 'Account temporarily locked',
        remainingMinutes: remainingTime
      });
    }

    // Find user
    const user = users.get(email);
    if (!user) {
      recordFailedAttempt(email);
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Verify password
    console.time(`Password verification for ${email}`);
    const validPassword = await bcrypt.compare(password, user.passwordHash);
    console.timeEnd(`Password verification for ${email}`);

    if (!validPassword) {
      recordFailedAttempt(email);
      const attempts = loginAttempts.get(email);
      const remainingAttempts = CONFIG.MAX_LOGIN_ATTEMPTS - attempts.count;
      
      return res.status(401).json({ 
        error: 'Invalid credentials',
        remainingAttempts
      });
    }

    // Reset login attempts on successful login
    resetLoginAttempts(email);

    // Update last login
    user.lastLogin = new Date();

    // Generate tokens
    const accessToken = generateAccessToken(user);
    const { token: refreshToken, tokenId } = generateRefreshToken(user);

    // Log authentication event
    console.log(`[AUTH] Login successful: ${email} (${user.role})`);

    res.json({
      message: 'Login successful',
      accessToken,
      refreshToken,
      tokenType: 'Bearer',
      expiresIn: 900, // 15 minutes in seconds
      user: {
        id: user.id,
        email: user.email,
        role: user.role,
        accountNumber: user.accountNumber
      }
    });

  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Route: Refresh access token
app.post('/auth/refresh', (req, res) => {
  try {
    const { refreshToken } = req.body;

    if (!refreshToken) {
      return res.status(400).json({ error: 'Refresh token required' });
    }

    // Verify refresh token
    jwt.verify(refreshToken, CONFIG.JWT_REFRESH_SECRET, (err, decoded) => {
      if (err) {
        return res.status(403).json({ error: 'Invalid refresh token' });
      }

      if (decoded.type !== 'refresh') {
        return res.status(403).json({ error: 'Invalid token type' });
      }

      // Check if refresh token exists in store
      const storedToken = refreshTokens.get(decoded.tokenId);
      if (!storedToken || storedToken.token !== refreshToken) {
        return res.status(403).json({ error: 'Refresh token revoked or invalid' });
      }

      // Find user
      const user = Array.from(users.values()).find(u => u.id === decoded.userId);
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }

      // Generate new access token
      const newAccessToken = generateAccessToken(user);

      console.log(`[AUTH] Token refreshed: ${user.email}`);

      res.json({
        accessToken: newAccessToken,
        tokenType: 'Bearer',
        expiresIn: 900
      });
    });

  } catch (error) {
    console.error('Token refresh error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Route: Logout
app.post('/auth/logout', authenticateToken, (req, res) => {
  try {
    const { refreshToken } = req.body;

    if (refreshToken) {
      // Decode to get token ID
      const decoded = jwt.decode(refreshToken);
      if (decoded && decoded.tokenId) {
        // Remove refresh token from store
        refreshTokens.delete(decoded.tokenId);
        console.log(`[AUTH] Refresh token revoked: ${decoded.tokenId}`);
      }
    }

    console.log(`[AUTH] Logout: ${req.user.email}`);

    res.json({ message: 'Logout successful' });

  } catch (error) {
    console.error('Logout error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Protected route: Get current user profile
app.get('/auth/profile', authenticateToken, (req, res) => {
  const user = users.get(req.user.email);
  
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }

  res.json({
    id: user.id,
    email: user.email,
    role: user.role,
    accountNumber: user.accountNumber,
    mfaEnabled: user.mfaEnabled,
    createdAt: user.createdAt,
    lastLogin: user.lastLogin
  });
});

// Protected route: Get account balance (customer only)
app.get('/api/account/balance', 
  authenticateToken,
  authorizeRole('customer', 'teller', 'manager'),
  (req, res) => {
    // Simulate balance check
    const balance = Math.random() * 10000;
    
    res.json({
      accountNumber: req.user.accountNumber,
      balance: balance.toFixed(2),
      currency: 'USD',
      asOf: new Date()
    });
  }
);

// Protected route: Admin dashboard (admin only)
app.get('/api/admin/dashboard',
  authenticateToken,
  authorizeRole('admin', 'manager'),
  (req, res) => {
    res.json({
      totalUsers: users.size,
      activeTokens: refreshTokens.size,
      lockedAccounts: Array.from(loginAttempts.values())
        .filter(a => a.lockedUntil && Date.now() < a.lockedUntil).length
    });
  }
);

// Start server
const PORT = process.env.PORT || 3000;

async function start() {
  await initializeUsers();
  
  app.listen(PORT, () => {
    console.log(`\n🔐 Authentication Service running on port ${PORT}`);
    console.log(`\nAccess Token Expiry: ${CONFIG.ACCESS_TOKEN_EXPIRY}`);
    console.log(`Refresh Token Expiry: ${CONFIG.REFRESH_TOKEN_EXPIRY}`);
    console.log(`Bcrypt Rounds: ${CONFIG.BCRYPT_ROUNDS}`);
    console.log(`\nTest with:\n`);
    console.log(`# Register new user`);
    console.log(`curl -X POST http://localhost:${PORT}/auth/register \\`);
    console.log(`  -H "Content-Type: application/json" \\`);
    console.log(`  -d '{"email":"test@bank.com","password":"Test123!","accountNumber":"ACC999"}'`);
    console.log(`\n# Login`);
    console.log(`curl -X POST http://localhost:${PORT}/auth/login \\`);
    console.log(`  -H "Content-Type: application/json" \\`);
    console.log(`  -d '{"email":"customer@bank.com","password":"Customer123!"}'`);
  });
}

start();

module.exports = { app, authenticateToken, authorizeRole };
```

### package.json

```json
{
  "name": "banking-auth-service",
  "version": "1.0.0",
  "description": "JWT Authentication for Banking API",
  "main": "auth-service.js",
  "scripts": {
    "start": "node auth-service.js",
    "dev": "nodemon auth-service.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "bcrypt": "^5.1.1",
    "jsonwebtoken": "^9.0.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

---

## 💻 Example 2: Session-Based Authentication with Express-Session

This example implements traditional session-based authentication with secure cookies.

```javascript
// session-auth.js
const express = require('express');
const session = require('express-session');
const bcrypt = require('bcrypt');
const crypto = require('crypto');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const app = express();
app.use(express.json());

// Redis client for session storage (in production)
// For development, we'll use memory store
const sessionConfig = {
  secret: process.env.SESSION_SECRET || crypto.randomBytes(64).toString('hex'),
  name: 'bankingSessionId', // Custom cookie name
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production', // HTTPS only in production
    httpOnly: true, // Prevents XSS attacks
    maxAge: 30 * 60 * 1000, // 30 minutes
    sameSite: 'strict' // CSRF protection
  }
};

// If Redis is available, use it for session storage
if (process.env.REDIS_URL) {
  const redisClient = createClient({ url: process.env.REDIS_URL });
  redisClient.connect().catch(console.error);
  sessionConfig.store = new RedisStore({ client: redisClient });
  console.log('Using Redis for session storage');
} else {
  console.log('Using memory store for sessions (development only)');
}

app.use(session(sessionConfig));

// Simulated database
const users = new Map();
const BCRYPT_ROUNDS = 12;

// Initialize sample users
async function initUsers() {
  const testUsers = [
    { username: 'john.doe', password: 'Customer123!', role: 'customer', accountNumber: 'ACC001', fullName: 'John Doe' },
    { username: 'jane.smith', password: 'Customer123!', role: 'customer', accountNumber: 'ACC002', fullName: 'Jane Smith' },
    { username: 'admin', password: 'Admin123!', role: 'admin', accountNumber: 'ADM001', fullName: 'Admin User' }
  ];

  for (const userData of testUsers) {
    const passwordHash = await bcrypt.hash(userData.password, BCRYPT_ROUNDS);
    users.set(userData.username, {
      id: crypto.randomUUID(),
      username: userData.username,
      passwordHash,
      role: userData.role,
      accountNumber: userData.accountNumber,
      fullName: userData.fullName,
      createdAt: new Date()
    });
  }

  console.log('Sample users created:');
  testUsers.forEach(u => console.log(`- ${u.username} / ${u.password} (${u.role})`));
}

// Middleware: Check if user is authenticated
function isAuthenticated(req, res, next) {
  if (req.session && req.session.userId) {
    return next();
  }
  res.status(401).json({ error: 'Authentication required' });
}

// Middleware: Check role
function hasRole(...roles) {
  return (req, res, next) => {
    if (!req.session || !req.session.userId) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const user = Array.from(users.values()).find(u => u.id === req.session.userId);
    
    if (!user || !roles.includes(user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    req.user = user;
    next();
  };
}

// Route: Login
app.post('/auth/login', async (req, res) => {
  try {
    const { username, password, rememberMe } = req.body;

    if (!username || !password) {
      return res.status(400).json({ error: 'Username and password required' });
    }

    // Find user
    const user = users.get(username);
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Verify password
    const validPassword = await bcrypt.compare(password, user.passwordHash);
    if (!validPassword) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Regenerate session ID to prevent session fixation
    req.session.regenerate((err) => {
      if (err) {
        return res.status(500).json({ error: 'Session creation failed' });
      }

      // Store user data in session
      req.session.userId = user.id;
      req.session.username = user.username;
      req.session.role = user.role;
      req.session.loginTime = new Date();

      // Extend cookie expiry if "remember me" is checked
      if (rememberMe) {
        req.session.cookie.maxAge = 30 * 24 * 60 * 60 * 1000; // 30 days
      }

      console.log(`[AUTH] Login successful: ${username} (Session: ${req.sessionID})`);

      res.json({
        message: 'Login successful',
        user: {
          id: user.id,
          username: user.username,
          fullName: user.fullName,
          role: user.role,
          accountNumber: user.accountNumber
        },
        sessionId: req.sessionID
      });
    });

  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Route: Logout
app.post('/auth/logout', isAuthenticated, (req, res) => {
  const username = req.session.username;
  const sessionId = req.sessionID;

  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ error: 'Logout failed' });
    }

    res.clearCookie('bankingSessionId');
    console.log(`[AUTH] Logout successful: ${username} (Session: ${sessionId})`);
    
    res.json({ message: 'Logout successful' });
  });
});

// Route: Get current session info
app.get('/auth/session', isAuthenticated, (req, res) => {
  const user = Array.from(users.values()).find(u => u.id === req.session.userId);

  res.json({
    sessionId: req.sessionID,
    user: {
      id: user.id,
      username: user.username,
      fullName: user.fullName,
      role: user.role,
      accountNumber: user.accountNumber
    },
    loginTime: req.session.loginTime,
    cookie: {
      maxAge: req.session.cookie.maxAge,
      expires: req.session.cookie.expires
    }
  });
});

// Protected route: Account balance
app.get('/api/balance', isAuthenticated, hasRole('customer', 'admin'), (req, res) => {
  const balance = (Math.random() * 50000).toFixed(2);
  
  res.json({
    accountNumber: req.user.accountNumber,
    balance,
    currency: 'USD',
    availableBalance: (balance * 0.9).toFixed(2)
  });
});

// Protected route: Transfer money
app.post('/api/transfer', isAuthenticated, hasRole('customer'), async (req, res) => {
  const { toAccount, amount } = req.body;

  // Validate
  if (!toAccount || !amount || amount <= 0) {
    return res.status(400).json({ error: 'Invalid transfer details' });
  }

  // For high-value transfers, require re-authentication
  if (amount > 10000) {
    const { password } = req.body;
    
    if (!password) {
      return res.status(403).json({ 
        error: 'Password confirmation required for high-value transfers',
        requiresPassword: true
      });
    }

    // Re-verify password
    const user = Array.from(users.values()).find(u => u.id === req.session.userId);
    const validPassword = await bcrypt.compare(password, user.passwordHash);
    
    if (!validPassword) {
      return res.status(401).json({ error: 'Invalid password' });
    }
  }

  // Process transfer (simulated)
  const transactionId = crypto.randomUUID();

  console.log(`[TRANSFER] ${req.user.accountNumber} → ${toAccount}: $${amount} (TXN: ${transactionId})`);

  res.json({
    message: 'Transfer successful',
    transactionId,
    from: req.user.accountNumber,
    to: toAccount,
    amount,
    timestamp: new Date()
  });
});

// Admin route: View all active sessions
app.get('/api/admin/sessions', isAuthenticated, hasRole('admin'), (req, res) => {
  // In production with Redis, you'd query Redis for all sessions
  res.json({
    message: 'Active sessions endpoint',
    note: 'Would query Redis store in production',
    currentSession: req.sessionID
  });
});

// Start server
const PORT = process.env.PORT || 3001;

async function start() {
  await initUsers();
  
  app.listen(PORT, () => {
    console.log(`\n🔐 Session-Based Auth Service running on port ${PORT}`);
    console.log(`\nTest commands:\n`);
    console.log(`# Login`);
    console.log(`curl -X POST http://localhost:${PORT}/auth/login \\`);
    console.log(`  -H "Content-Type: application/json" \\`);
    console.log(`  -d '{"username":"john.doe","password":"Customer123!"}' \\`);
    console.log(`  -c cookies.txt  # Save cookies`);
    console.log(`\n# Check balance (with session cookie)`);
    console.log(`curl http://localhost:${PORT}/api/balance \\`);
    console.log(`  -b cookies.txt  # Send cookies`);
    console.log(`\n# Logout`);
    console.log(`curl -X POST http://localhost:${PORT}/auth/logout \\`);
    console.log(`  -b cookies.txt`);
  });
}

start();
```

---

## 💻 Example 3: Multi-Factor Authentication (MFA) and OAuth 2.0

This example adds MFA using TOTP (Time-based One-Time Password) and demonstrates OAuth 2.0 integration.

```javascript
// mfa-oauth-auth.js
const express = require('express');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');
const crypto = require('crypto');

const app = express();
app.use(express.json());

// Configuration
const CONFIG = {
  JWT_SECRET: process.env.JWT_SECRET || crypto.randomBytes(64).toString('hex'),
  BCRYPT_ROUNDS: 12,
  MFA_ISSUER: 'SecureBank',
  MFA_WINDOW: 2 // Allow 2 time windows (60 seconds each)
};

// Simulated database
const users = new Map();
const pendingMfaVerifications = new Map();

// User model with MFA support
class User {
  constructor(id, email, passwordHash, accountNumber) {
    this.id = id;
    this.email = email;
    this.passwordHash = passwordHash;
    this.accountNumber = accountNumber;
    this.role = 'customer';
    this.mfaEnabled = false;
    this.mfaSecret = null;
    this.mfaBackupCodes = [];
    this.oauthProviders = {}; // Store OAuth provider IDs
  }
}

// Initialize sample user
async function initUsers() {
  const hash = await bcrypt.hash('Customer123!', CONFIG.BCRYPT_ROUNDS);
  const user = new User('user-001', 'alice@bank.com', hash, 'ACC001');
  users.set(user.email, user);
  console.log('Sample user: alice@bank.com / Customer123!');
}

// Helper: Generate backup codes
function generateBackupCodes(count = 10) {
  const codes = [];
  for (let i = 0; i < count; i++) {
    codes.push(crypto.randomBytes(4).toString('hex').toUpperCase());
  }
  return codes;
}

// Route: Setup MFA
app.post('/auth/mfa/setup', async (req, res) => {
  try {
    const { email, password } = req.body;

    // Verify user credentials first
    const user = users.get(email);
    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    if (user.mfaEnabled) {
      return res.status(400).json({ error: 'MFA already enabled' });
    }

    // Generate MFA secret
    const secret = speakeasy.generateSecret({
      name: `${CONFIG.MFA_ISSUER} (${user.email})`,
      issuer: CONFIG.MFA_ISSUER,
      length: 32
    });

    // Generate backup codes
    const backupCodes = generateBackupCodes();

    // Store temporarily (not saved until verified)
    pendingMfaVerifications.set(user.id, {
      secret: secret.base32,
      backupCodes,
      timestamp: Date.now()
    });

    // Generate QR code
    const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url);

    console.log(`[MFA] Setup initiated for ${user.email}`);

    res.json({
      message: 'Scan QR code with authenticator app',
      qrCode: qrCodeUrl,
      manualEntryCode: secret.base32,
      backupCodes,
      note: 'Save backup codes in a secure location'
    });

  } catch (error) {
    console.error('MFA setup error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Route: Verify and enable MFA
app.post('/auth/mfa/verify', async (req, res) => {
  try {
    const { email, token } = req.body;

    const user = users.get(email);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    const pending = pendingMfaVerifications.get(user.id);
    if (!pending) {
      return res.status(400).json({ error: 'No pending MFA setup' });
    }

    // Verify token
    const verified = speakeasy.totp.verify({
      secret: pending.secret,
      encoding: 'base32',
      token,
      window: CONFIG.MFA_WINDOW
    });

    if (!verified) {
      return res.status(401).json({ error: 'Invalid MFA token' });
    }

    // Enable MFA for user
    user.mfaEnabled = true;
    user.mfaSecret = pending.secret;
    user.mfaBackupCodes = pending.backupCodes;

    // Clear pending verification
    pendingMfaVerifications.delete(user.id);

    console.log(`[MFA] Enabled for ${user.email}`);

    res.json({
      message: 'MFA enabled successfully',
      backupCodes: user.mfaBackupCodes
    });

  } catch (error) {
    console.error('MFA verification error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Route: Login with MFA
app.post('/auth/login', async (req, res) => {
  try {
    const { email, password, mfaToken, backupCode } = req.body;

    // Verify credentials
    const user = users.get(email);
    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // If MFA is enabled, require MFA token
    if (user.mfaEnabled) {
      if (!mfaToken && !backupCode) {
        return res.status(403).json({ 
          error: 'MFA token required',
          mfaRequired: true
        });
      }

      let mfaVerified = false;

      // Verify MFA token
      if (mfaToken) {
        mfaVerified = speakeasy.totp.verify({
          secret: user.mfaSecret,
          encoding: 'base32',
          token: mfaToken,
          window: CONFIG.MFA_WINDOW
        });
      }

      // Verify backup code
      if (!mfaVerified && backupCode) {
        const codeIndex = user.mfaBackupCodes.indexOf(backupCode.toUpperCase());
        if (codeIndex !== -1) {
          // Remove used backup code
          user.mfaBackupCodes.splice(codeIndex, 1);
          mfaVerified = true;
          console.log(`[MFA] Backup code used for ${user.email}. Remaining: ${user.mfaBackupCodes.length}`);
        }
      }

      if (!mfaVerified) {
        return res.status(401).json({ error: 'Invalid MFA token' });
      }
    }

    // Generate JWT
    const token = jwt.sign(
      { userId: user.id, email: user.email, role: user.role },
      CONFIG.JWT_SECRET,
      { expiresIn: '1h' }
    );

    console.log(`[AUTH] Login successful: ${user.email} (MFA: ${user.mfaEnabled})`);

    res.json({
      message: 'Login successful',
      token,
      user: {
        id: user.id,
        email: user.email,
        accountNumber: user.accountNumber,
        mfaEnabled: user.mfaEnabled
      }
    });

  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Route: Disable MFA
app.post('/auth/mfa/disable', async (req, res) => {
  try {
    const { email, password, mfaToken } = req.body;

    const user = users.get(email);
    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    if (!user.mfaEnabled) {
      return res.status(400).json({ error: 'MFA not enabled' });
    }

    // Verify current MFA token before disabling
    const verified = speakeasy.totp.verify({
      secret: user.mfaSecret,
      encoding: 'base32',
      token: mfaToken,
      window: CONFIG.MFA_WINDOW
    });

    if (!verified) {
      return res.status(401).json({ error: 'Invalid MFA token' });
    }

    // Disable MFA
    user.mfaEnabled = false;
    user.mfaSecret = null;
    user.mfaBackupCodes = [];

    console.log(`[MFA] Disabled for ${user.email}`);

    res.json({ message: 'MFA disabled successfully' });

  } catch (error) {
    console.error('MFA disable error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// OAuth 2.0 Simulation (GitHub-like flow)
const oauthClients = new Map();
const authorizationCodes = new Map();

// Route: OAuth - Authorization endpoint
app.get('/oauth/authorize', (req, res) => {
  const { client_id, redirect_uri, state, scope } = req.query;

  if (!client_id || !redirect_uri) {
    return res.status(400).json({ error: 'Missing required parameters' });
  }

  // In production, show consent screen
  // For demo, auto-approve and generate code
  const code = crypto.randomBytes(32).toString('hex');
  
  authorizationCodes.set(code, {
    clientId: client_id,
    redirectUri: redirect_uri,
    userId: 'user-001', // Simulated logged-in user
    scope: scope || 'read:account',
    expiresAt: Date.now() + 10 * 60 * 1000 // 10 minutes
  });

  // Redirect with authorization code
  const redirectUrl = `${redirect_uri}?code=${code}&state=${state || ''}`;
  
  console.log(`[OAUTH] Authorization code generated for client ${client_id}`);
  
  res.json({
    message: 'In production, redirect to',
    redirect: redirectUrl,
    code // Shown for testing only
  });
});

// Route: OAuth - Token endpoint
app.post('/oauth/token', (req, res) => {
  const { grant_type, code, client_id, client_secret, redirect_uri } = req.body;

  if (grant_type !== 'authorization_code') {
    return res.status(400).json({ error: 'Unsupported grant type' });
  }

  const authCode = authorizationCodes.get(code);
  
  if (!authCode) {
    return res.status(401).json({ error: 'Invalid authorization code' });
  }

  if (authCode.expiresAt < Date.now()) {
    authorizationCodes.delete(code);
    return res.status(401).json({ error: 'Authorization code expired' });
  }

  if (authCode.clientId !== client_id || authCode.redirectUri !== redirect_uri) {
    return res.status(401).json({ error: 'Invalid client or redirect URI' });
  }

  // Delete used code (one-time use)
  authorizationCodes.delete(code);

  // Generate access token
  const accessToken = jwt.sign(
    { userId: authCode.userId, clientId: client_id, scope: authCode.scope },
    CONFIG.JWT_SECRET,
    { expiresIn: '1h' }
  );

  console.log(`[OAUTH] Access token issued for client ${client_id}`);

  res.json({
    access_token: accessToken,
    token_type: 'Bearer',
    expires_in: 3600,
    scope: authCode.scope
  });
});

// Route: OAuth - Resource endpoint (protected by OAuth token)
app.get('/oauth/userinfo', (req, res) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  jwt.verify(token, CONFIG.JWT_SECRET, (err, decoded) => {
    if (err) {
      return res.status(403).json({ error: 'Invalid token' });
    }

    const user = Array.from(users.values()).find(u => u.id === decoded.userId);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Return user info based on scope
    res.json({
      id: user.id,
      email: user.email,
      accountNumber: user.accountNumber,
      scope: decoded.scope
    });
  });
});

// Start server
const PORT = process.env.PORT || 3002;

async function start() {
  await initUsers();
  
  app.listen(PORT, () => {
    console.log(`\n🔐 MFA & OAuth Auth Service running on port ${PORT}`);
    console.log(`\nMFA Setup Flow:`);
    console.log(`1. POST /auth/mfa/setup - Get QR code`);
    console.log(`2. Scan with Google Authenticator / Authy`);
    console.log(`3. POST /auth/mfa/verify - Confirm setup`);
    console.log(`4. POST /auth/login - Login with MFA token`);
    console.log(`\nOAuth Flow:`);
    console.log(`1. GET /oauth/authorize - Get auth code`);
    console.log(`2. POST /oauth/token - Exchange for access token`);
    console.log(`3. GET /oauth/userinfo - Access protected resource`);
  });
}

start();
```

### package.json for MFA example

```json
{
  "name": "banking-mfa-oauth",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2",
    "bcrypt": "^5.1.1",
    "jsonwebtoken": "^9.0.2",
    "speakeasy": "^2.0.0",
    "qrcode": "^1.5.3"
  }
}
```

---

## 🧪 Testing Instructions

### Setup

```bash
mkdir banking-auth-examples
cd banking-auth-examples

# Create three example projects
mkdir jwt-auth session-auth mfa-auth
cd jwt-auth
npm init -y
npm install express bcrypt jsonwebtoken
# Copy Example 1 code

cd ../session-auth
npm init -y
npm install express bcrypt express-session connect-redis redis
# Copy Example 2 code

cd ../mfa-auth
npm init -y
npm install express bcrypt jsonwebtoken speakeasy qrcode
# Copy Example 3 code
```

### Test Example 1: JWT Authentication

```bash
cd jwt-auth
node auth-service.js

# Terminal 2: Test registration
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@bank.com","password":"Test123!","accountNumber":"ACC999"}'

# Test login
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"customer@bank.com","password":"Customer123!"}'

# Save the tokens from response
ACCESS_TOKEN="<access_token_from_response>"
REFRESH_TOKEN="<refresh_token_from_response>"

# Access protected route
curl http://localhost:3000/auth/profile \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Get account balance
curl http://localhost:3000/api/account/balance \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Wait 16 minutes for token to expire, then refresh
curl -X POST http://localhost:3000/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH_TOKEN\"}"

# Logout
curl -X POST http://localhost:3000/auth/logout \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH_TOKEN\"}"
```

### Test Example 2: Session Authentication

```bash
cd session-auth
node session-auth.js

# Login and save cookies
curl -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john.doe","password":"Customer123!"}' \
  -c cookies.txt

# Check session
curl http://localhost:3001/auth/session \
  -b cookies.txt

# Access protected route
curl http://localhost:3001/api/balance \
  -b cookies.txt

# Transfer money (small amount)
curl -X POST http://localhost:3001/api/transfer \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"toAccount":"ACC999","amount":500}'

# Transfer money (high value - requires password)
curl -X POST http://localhost:3001/api/transfer \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"toAccount":"ACC999","amount":15000,"password":"Customer123!"}'

# Logout
curl -X POST http://localhost:3001/auth/logout \
  -b cookies.txt
```

### Test Example 3: MFA Authentication

```bash
cd mfa-auth
node mfa-oauth-auth.js

# Step 1: Setup MFA
curl -X POST http://localhost:3002/auth/mfa/setup \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@bank.com","password":"Customer123!"}'

# Response includes QR code (data URL) and backup codes
# Scan QR code with Google Authenticator or Authy

# Step 2: Verify MFA setup with 6-digit token from app
curl -X POST http://localhost:3002/auth/mfa/verify \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@bank.com","token":"123456"}'  # Use actual token

# Step 3: Login with MFA
curl -X POST http://localhost:3002/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@bank.com","password":"Customer123!","mfaToken":"123456"}'

# Test backup code login
curl -X POST http://localhost:3002/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@bank.com","password":"Customer123!","backupCode":"ABCD1234"}'

# Disable MFA
curl -X POST http://localhost:3002/auth/mfa/disable \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@bank.com","password":"Customer123!","mfaToken":"123456"}'

# OAuth Flow
# Step 1: Get authorization code
curl "http://localhost:3002/oauth/authorize?client_id=app123&redirect_uri=http://localhost/callback&state=xyz"

# Step 2: Exchange code for token
CODE="<code_from_step1>"
curl -X POST http://localhost:3002/oauth/token \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"authorization_code\",\"code\":\"$CODE\",\"client_id\":\"app123\",\"redirect_uri\":\"http://localhost/callback\"}"

# Step 3: Access protected resource
TOKEN="<token_from_step2>"
curl http://localhost:3002/oauth/userinfo \
  -H "Authorization: Bearer $TOKEN"
```

---

## ✅ DO's

1. **DO** use bcrypt with cost factor ≥ 12 for password hashing
2. **DO** implement short-lived access tokens (15 minutes) with refresh tokens
3. **DO** use HTTPS only in production (secure cookie flag)
4. **DO** set httpOnly and sameSite flags on cookies
5. **DO** implement rate limiting on login endpoints
6. **DO** log all authentication events for audit trails
7. **DO** use strong, random secrets for JWT signing (64+ bytes)
8. **DO** implement account lockout after failed attempts
9. **DO** require MFA for high-value transactions
10. **DO** store refresh tokens securely (Redis with TTL)

---

## ❌ DON'Ts

1. **DON'T** store passwords in plain text (always hash)
2. **DON'T** use MD5 or SHA1 for passwords (use bcrypt/argon2)
3. **DON'T** expose JWT secrets in client-side code
4. **DON'T** store tokens in localStorage (use httpOnly cookies or memory)
5. **DON'T** use long-lived access tokens (>1 hour)
6. **DON'T** skip CSRF protection for session-based auth
7. **DON'T** allow unlimited login attempts (implement rate limiting)
8. **DON'T** log sensitive data (passwords, tokens)
9. **DON'T** trust client-side validation only
10. **DON'T** use weak password policies (require 8+ chars, complexity)

---

## 🎯 Key Takeaways

1. **JWT vs Sessions**: JWT is stateless and scales better; sessions offer easier revocation
2. **Token Lifetime**: Short access tokens (15min) + long refresh tokens (7d)
3. **Password Security**: bcrypt with cost factor 12 = ~260ms hashing time
4. **MFA**: Adds critical security layer for banking applications
5. **Cookie Security**: httpOnly + secure + sameSite = essential flags
6. **Rate Limiting**: Prevent brute force attacks on login endpoints
7. **Audit Logging**: Log all auth events for compliance and forensics
8. **OAuth 2.0**: Use authorization code flow for third-party integrations
9. **Token Storage**: Never store sensitive tokens in localStorage
10. **Production Security**: Always use HTTPS, strong secrets, and proper monitoring

**Banking Security Requirements**:
- PCI DSS compliance for cardholder data
- SOC 2 Type II for service providers
- Multi-factor authentication for high-risk operations
- Comprehensive audit trails
- Regular security assessments and penetration testing

---

**File**: `Q27_Security_Authentication.md`  
**Status**: ✅ Complete with 3 production-ready banking examples
