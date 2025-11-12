# Security Questions 16-20: Attack Prevention & Security Headers

---

### Q16. How do you prevent XSS (Cross-Site Scripting) attacks?

**Answer:**

```javascript
const express = require('express');
const helmet = require('helmet');
const DOMPurify = require('isomorphic-dompurify');
const validator = require('validator');

/**
 * XSS Prevention Service
 */
class XSSPreventionService {
  /**
   * Sanitize user input
   */
  sanitizeInput(input, options = {}) {
    if (typeof input !== 'string') {
      return input;
    }
    
    const {
      allowHTML = false,
      allowedTags = [],
      stripTags = true
    } = options;
    
    if (!allowHTML) {
      // Remove all HTML tags
      return validator.stripLow(validator.escape(input));
    }
    
    // Allow specific HTML tags
    return DOMPurify.sanitize(input, {
      ALLOWED_TAGS: allowedTags,
      ALLOWED_ATTR: ['href', 'src', 'alt', 'title'],
      KEEP_CONTENT: true
    });
  }
  
  /**
   * Sanitize object recursively
   */
  sanitizeObject(obj, options = {}) {
    if (typeof obj !== 'object' || obj === null) {
      return this.sanitizeInput(obj, options);
    }
    
    if (Array.isArray(obj)) {
      return obj.map(item => this.sanitizeObject(item, options));
    }
    
    const sanitized = {};
    for (const [key, value] of Object.entries(obj)) {
      sanitized[key] = this.sanitizeObject(value, options);
    }
    
    return sanitized;
  }
  
  /**
   * Input sanitization middleware
   */
  sanitizeMiddleware(options = {}) {
    return (req, res, next) => {
      if (req.body) {
        req.body = this.sanitizeObject(req.body, options);
      }
      
      if (req.query) {
        req.query = this.sanitizeObject(req.query, options);
      }
      
      if (req.params) {
        req.params = this.sanitizeObject(req.params, options);
      }
      
      next();
    };
  }
  
  /**
   * Output encoding for different contexts
   */
  encodeForHTML(str) {
    const htmlEntities = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#x27;',
      '/': '&#x2F;'
    };
    
    return str.replace(/[&<>"'/]/g, (char) => htmlEntities[char]);
  }
  
  encodeForJavaScript(str) {
    return str
      .replace(/\\/g, '\\\\')
      .replace(/'/g, "\\'")
      .replace(/"/g, '\\"')
      .replace(/</g, '\\x3C')
      .replace(/>/g, '\\x3E')
      .replace(/\n/g, '\\n')
      .replace(/\r/g, '\\r');
  }
  
  encodeForURL(str) {
    return encodeURIComponent(str);
  }
  
  encodeForCSS(str) {
    return str.replace(/[^a-zA-Z0-9]/g, (char) => {
      return '\\' + char.charCodeAt(0).toString(16) + ' ';
    });
  }
  
  /**
   * Validate URL to prevent javascript: protocol
   */
  sanitizeURL(url) {
    const dangerousProtocols = ['javascript:', 'data:', 'vbscript:'];
    
    const lowerURL = url.toLowerCase().trim();
    
    for (const protocol of dangerousProtocols) {
      if (lowerURL.startsWith(protocol)) {
        return '#'; // Safe fallback
      }
    }
    
    // Validate URL format
    if (!validator.isURL(url, { protocols: ['http', 'https'], require_protocol: true })) {
      return '#';
    }
    
    return url;
  }
}

/**
 * Content Security Policy Configuration
 */
class CSPService {
  /**
   * Get CSP middleware
   */
  static getCSPMiddleware() {
    return helmet.contentSecurityPolicy({
      directives: {
        defaultSrc: ["'self'"],
        
        scriptSrc: [
          "'self'",
          "'nonce-{{nonce}}'", // Dynamic nonce
          "https://cdn.enbd.com"
        ],
        
        styleSrc: [
          "'self'",
          "'nonce-{{nonce}}'",
          "https://cdn.enbd.com"
        ],
        
        imgSrc: [
          "'self'",
          "data:",
          "https:"
        ],
        
        fontSrc: [
          "'self'",
          "https://fonts.gstatic.com"
        ],
        
        connectSrc: [
          "'self'",
          "https://api.enbd.com"
        ],
        
        frameSrc: ["'none'"],
        
        objectSrc: ["'none'"],
        
        baseUri: ["'self'"],
        
        formAction: ["'self'"],
        
        frameAncestors: ["'none'"],
        
        upgradeInsecureRequests: []
      },
      
      reportOnly: false
    });
  }
  
  /**
   * Generate nonce for inline scripts
   */
  static generateNonce() {
    return crypto.randomBytes(16).toString('base64');
  }
  
  /**
   * Nonce middleware
   */
  static nonceMiddleware() {
    return (req, res, next) => {
      res.locals.nonce = CSPService.generateNonce();
      next();
    };
  }
}

/**
 * Secure Template Rendering
 */
class SecureTemplateService {
  /**
   * Render template with auto-escaping
   */
  render(template, data) {
    const xssService = new XSSPreventionService();
    
    // Escape all data by default
    const escapedData = {};
    for (const [key, value] of Object.entries(data)) {
      if (typeof value === 'string') {
        escapedData[key] = xssService.encodeForHTML(value);
      } else {
        escapedData[key] = value;
      }
    }
    
    // Replace template placeholders
    return template.replace(/{{(\w+)}}/g, (match, key) => {
      return escapedData[key] || '';
    });
  }
  
  /**
   * Render with raw HTML (use carefully)
   */
  renderRaw(template, data) {
    return template.replace(/{{(\w+)}}/g, (match, key) => {
      return data[key] || '';
    });
  }
}

/**
 * Complete XSS Prevention Application
 */
class XSSProtectedApplication {
  constructor() {
    this.app = express();
    this.xssService = new XSSPreventionService();
    this.templateService = new SecureTemplateService();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    // JSON body parser
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
    
    // Security headers
    this.app.use(helmet());
    
    // CSP with nonce
    this.app.use(CSPService.nonceMiddleware());
    this.app.use(CSPService.getCSPMiddleware());
    
    // Input sanitization
    this.app.use(this.xssService.sanitizeMiddleware());
    
    // Additional security headers
    this.app.use((req, res, next) => {
      res.setHeader('X-Content-Type-Options', 'nosniff');
      res.setHeader('X-Frame-Options', 'DENY');
      res.setHeader('X-XSS-Protection', '1; mode=block');
      next();
    });
  }
  
  setupRoutes() {
    // Profile page (vulnerable if not handled)
    this.app.get('/profile', async (req, res) => {
      const user = await this.getUser(req.query.id);
      
      // Safe rendering with automatic escaping
      const html = this.templateService.render(`
        <html>
          <head>
            <title>ENBD - Profile</title>
            <script nonce="${res.locals.nonce}">
              console.log('Secure script');
            </script>
          </head>
          <body>
            <h1>Profile: {{name}}</h1>
            <p>Email: {{email}}</p>
            <p>Bio: {{bio}}</p>
          </body>
        </html>
      `, {
        name: user.name,
        email: user.email,
        bio: user.bio
      });
      
      res.send(html);
    });
    
    // Comment submission
    this.app.post('/api/comments', async (req, res) => {
      const { text, author } = req.body;
      
      // Input is already sanitized by middleware
      
      // Store comment
      await this.storeComment({
        text,
        author,
        createdAt: new Date()
      });
      
      res.json({ success: true });
    });
    
    // Search endpoint
    this.app.get('/api/search', async (req, res) => {
      const query = req.query.q;
      
      // Sanitize search query
      const sanitized = this.xssService.sanitizeInput(query);
      
      const results = await this.search(sanitized);
      
      res.json({ results, query: sanitized });
    });
    
    // Rich text editor
    this.app.post('/api/content', async (req, res) => {
      const { content } = req.body;
      
      // Allow specific HTML tags
      const sanitized = this.xssService.sanitizeInput(content, {
        allowHTML: true,
        allowedTags: ['b', 'i', 'u', 'p', 'br', 'a']
      });
      
      await this.saveContent(sanitized);
      
      res.json({ success: true });
    });
  }
  
  async getUser(id) {
    return {
      id,
      name: 'Ahmed Al Mansoori',
      email: 'ahmed@enbd.com',
      bio: 'Banking professional <script>alert("XSS")</script>'
    };
  }
  
  async storeComment(comment) {
    // Database storage
  }
  
  async search(query) {
    return [];
  }
  
  async saveContent(content) {
    // Database storage
  }
}

/**
 * React Component XSS Prevention
 */
const ReactXSSExample = `
// React automatically escapes content
function UserProfile({ user }) {
  // Safe - React escapes automatically
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.bio}</p>
    </div>
  );
}

// DANGEROUS - Using dangerouslySetInnerHTML
function RichContent({ html }) {
  // Must sanitize before using
  const sanitized = DOMPurify.sanitize(html);
  
  return (
    <div dangerouslySetInnerHTML={{ __html: sanitized }} />
  );
}

// Safe URL handling
function UserLink({ url }) {
  const sanitizeURL = (url) => {
    if (url.startsWith('javascript:')) {
      return '#';
    }
    return url;
  };
  
  return <a href={sanitizeURL(url)}>Link</a>;
}
`;

module.exports = {
  XSSPreventionService,
  CSPService,
  SecureTemplateService,
  XSSProtectedApplication
};
```

**ENBD XSS Prevention Examples:**

```javascript
// ❌ VULNERABLE
app.get('/profile', (req, res) => {
  const name = req.query.name;
  res.send(`<h1>Welcome ${name}</h1>`); // XSS!
});

// ✅ SAFE
app.get('/profile', (req, res) => {
  const xssService = new XSSPreventionService();
  const name = xssService.encodeForHTML(req.query.name);
  res.send(`<h1>Welcome ${name}</h1>`);
});

// ❌ VULNERABLE
const userInput = "<img src=x onerror=alert('XSS')>";
document.getElementById('content').innerHTML = userInput;

// ✅ SAFE
const sanitized = DOMPurify.sanitize(userInput);
document.getElementById('content').innerHTML = sanitized;
```

---

### Q17. How do you prevent CSRF (Cross-Site Request Forgery) attacks?

**Answer:**

```javascript
const crypto = require('crypto');
const csrf = require('csurf');

/**
 * CSRF Protection Service
 */
class CSRFProtectionService {
  constructor() {
    this.tokens = new Map();
  }
  
  /**
   * Generate CSRF token
   */
  generateToken(sessionId) {
    const token = crypto.randomBytes(32).toString('hex');
    
    // Store token with session
    this.tokens.set(sessionId, {
      token,
      createdAt: Date.now()
    });
    
    return token;
  }
  
  /**
   * Verify CSRF token
   */
  verifyToken(sessionId, token) {
    const stored = this.tokens.get(sessionId);
    
    if (!stored) {
      return false;
    }
    
    // Check expiration (1 hour)
    if (Date.now() - stored.createdAt > 3600000) {
      this.tokens.delete(sessionId);
      return false;
    }
    
    // Constant-time comparison
    return crypto.timingSafeEqual(
      Buffer.from(stored.token),
      Buffer.from(token)
    );
  }
  
  /**
   * Double Submit Cookie Pattern
   */
  generateDoubleCookie(res) {
    const token = crypto.randomBytes(32).toString('hex');
    
    // Set cookie
    res.cookie('csrf-token', token, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict'
    });
    
    return token;
  }
  
  /**
   * Verify double submit cookie
   */
  verifyDoubleCookie(cookieToken, headerToken) {
    if (!cookieToken || !headerToken) {
      return false;
    }
    
    return crypto.timingSafeEqual(
      Buffer.from(cookieToken),
      Buffer.from(headerToken)
    );
  }
  
  /**
   * Synchronizer Token Pattern Middleware
   */
  synchronizerTokenMiddleware() {
    return (req, res, next) => {
      // Skip for safe methods
      if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        return next();
      }
      
      const token = req.headers['x-csrf-token'] || 
                   req.body._csrf ||
                   req.query._csrf;
      
      if (!token) {
        return res.status(403).json({
          error: 'CSRF token missing'
        });
      }
      
      if (!this.verifyToken(req.sessionID, token)) {
        return res.status(403).json({
          error: 'Invalid CSRF token'
        });
      }
      
      next();
    };
  }
  
  /**
   * Double Submit Cookie Middleware
   */
  doubleSubmitMiddleware() {
    return (req, res, next) => {
      // Skip for safe methods
      if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        return next();
      }
      
      const cookieToken = req.cookies['csrf-token'];
      const headerToken = req.headers['x-csrf-token'];
      
      if (!this.verifyDoubleCookie(cookieToken, headerToken)) {
        return res.status(403).json({
          error: 'Invalid CSRF token'
        });
      }
      
      next();
    };
  }
  
  /**
   * SameSite Cookie Configuration
   */
  static configureSameSiteCookies(res) {
    res.cookie('session', 'value', {
      httpOnly: true,
      secure: true,
      sameSite: 'strict' // or 'lax' for some cross-site scenarios
    });
  }
  
  /**
   * Custom Header Verification
   */
  static customHeaderMiddleware() {
    return (req, res, next) => {
      // Skip for safe methods
      if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        return next();
      }
      
      // Verify custom header exists
      const customHeader = req.headers['x-requested-with'];
      
      if (customHeader !== 'XMLHttpRequest') {
        return res.status(403).json({
          error: 'Missing required header'
        });
      }
      
      next();
    };
  }
  
  /**
   * Origin/Referer Verification
   */
  static originVerificationMiddleware() {
    const allowedOrigins = [
      'https://app.enbd.com',
      'https://mobile.enbd.com'
    ];
    
    return (req, res, next) => {
      // Skip for safe methods
      if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        return next();
      }
      
      const origin = req.headers.origin || req.headers.referer;
      
      if (!origin) {
        return res.status(403).json({
          error: 'Missing origin header'
        });
      }
      
      const originURL = new URL(origin);
      const originBase = `${originURL.protocol}//${originURL.host}`;
      
      if (!allowedOrigins.includes(originBase)) {
        return res.status(403).json({
          error: 'Invalid origin'
        });
      }
      
      next();
    };
  }
}

/**
 * Complete CSRF Protected Application
 */
class CSRFProtectedApplication {
  constructor() {
    this.app = express();
    this.csrfService = new CSRFProtectionService();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
    this.app.use(require('cookie-parser')());
    
    // Session middleware
    this.app.use(session({
      secret: 'secret',
      resave: false,
      saveUninitialized: false,
      cookie: {
        httpOnly: true,
        secure: true,
        sameSite: 'strict'
      }
    }));
    
    // CSRF protection
    this.app.use(this.csrfService.synchronizerTokenMiddleware());
    
    // Origin verification
    this.app.use(CSRFProtectionService.originVerificationMiddleware());
  }
  
  setupRoutes() {
    // Get CSRF token
    this.app.get('/api/csrf-token', (req, res) => {
      const token = this.csrfService.generateToken(req.sessionID);
      res.json({ csrfToken: token });
    });
    
    // Login
    this.app.post('/api/auth/login', async (req, res) => {
      const { email, password } = req.body;
      
      // Authenticate user
      const user = await this.authenticateUser(email, password);
      
      if (!user) {
        return res.status(401).json({ error: 'Invalid credentials' });
      }
      
      // Create session
      req.session.userId = user.id;
      
      // Generate new CSRF token
      const csrfToken = this.csrfService.generateToken(req.sessionID);
      
      res.json({
        success: true,
        csrfToken,
        user
      });
    });
    
    // Transfer money (protected)
    this.app.post('/api/transfer', async (req, res) => {
      const { from, to, amount } = req.body;
      
      // Process transfer
      await this.processTransfer({
        userId: req.session.userId,
        from,
        to,
        amount
      });
      
      res.json({ success: true });
    });
    
    // Change password (protected)
    this.app.post('/api/change-password', async (req, res) => {
      const { oldPassword, newPassword } = req.body;
      
      await this.changePassword(
        req.session.userId,
        oldPassword,
        newPassword
      );
      
      res.json({ success: true });
    });
  }
  
  async authenticateUser(email, password) {
    return { id: 'user-123', email };
  }
  
  async processTransfer(data) {
    // Database operation
  }
  
  async changePassword(userId, oldPassword, newPassword) {
    // Database operation
  }
}

/**
 * Frontend Implementation
 */
const FrontendCSRFExample = `
// React component with CSRF protection
class TransferForm extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      csrfToken: null
    };
  }
  
  async componentDidMount() {
    // Get CSRF token
    const response = await fetch('/api/csrf-token');
    const data = await response.json();
    this.setState({ csrfToken: data.csrfToken });
  }
  
  async handleSubmit(e) {
    e.preventDefault();
    
    const formData = {
      from: this.state.from,
      to: this.state.to,
      amount: this.state.amount
    };
    
    // Include CSRF token in request
    const response = await fetch('/api/transfer', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': this.state.csrfToken,
        'X-Requested-With': 'XMLHttpRequest'
      },
      credentials: 'include',
      body: JSON.stringify(formData)
    });
    
    if (response.ok) {
      alert('Transfer successful');
    }
  }
  
  render() {
    return (
      <form onSubmit={this.handleSubmit}>
        {/* Form fields */}
      </form>
    );
  }
}

// Axios interceptor for CSRF
axios.interceptors.request.use(async (config) => {
  if (['post', 'put', 'delete', 'patch'].includes(config.method)) {
    // Get CSRF token from cookie or meta tag
    const token = document.querySelector('meta[name="csrf-token"]')?.content;
    
    if (token) {
      config.headers['X-CSRF-Token'] = token;
    }
  }
  
  return config;
});
`;

module.exports = {
  CSRFProtectionService,
  CSRFProtectedApplication
};
```

**CSRF Prevention Methods:**

1. **Synchronizer Token Pattern** ✅
2. **Double Submit Cookie** ✅
3. **SameSite Cookie Attribute** ✅
4. **Custom Request Headers** ✅
5. **Origin/Referer Verification** ✅

---

### Q18. How do you implement rate limiting and DDoS protection?

**Answer:**

```javascript
const Redis = require('ioredis');
const express = require('express');

/**
 * Rate Limiting Service
 */
class RateLimitingService {
  constructor() {
    this.redis = new Redis();
  }
  
  /**
   * Token Bucket Algorithm
   */
  async tokenBucket(key, capacity, refillRate) {
    const now = Date.now();
    const bucketKey = `ratelimit:tokenbucket:${key}`;
    
    // Get bucket state
    const bucket = await this.redis.hgetall(bucketKey);
    
    let tokens = parseFloat(bucket.tokens) || capacity;
    let lastRefill = parseInt(bucket.lastRefill) || now;
    
    // Refill tokens
    const timePassed = (now - lastRefill) / 1000;
    const tokensToAdd = timePassed * refillRate;
    tokens = Math.min(capacity, tokens + tokensToAdd);
    
    // Check if request can be allowed
    if (tokens < 1) {
      const waitTime = (1 - tokens) / refillRate;
      return {
        allowed: false,
        tokensRemaining: tokens,
        retryAfter: Math.ceil(waitTime)
      };
    }
    
    // Consume token
    tokens -= 1;
    
    // Update bucket
    await this.redis.hmset(bucketKey, {
      tokens: tokens.toString(),
      lastRefill: now.toString()
    });
    
    await this.redis.expire(bucketKey, 3600);
    
    return {
      allowed: true,
      tokensRemaining: Math.floor(tokens)
    };
  }
  
  /**
   * Sliding Window Algorithm
   */
  async slidingWindow(key, limit, windowMs) {
    const now = Date.now();
    const windowKey = `ratelimit:sliding:${key}`;
    
    // Remove old entries
    await this.redis.zremrangebyscore(
      windowKey,
      '-inf',
      now - windowMs
    );
    
    // Count requests in window
    const count = await this.redis.zcard(windowKey);
    
    if (count >= limit) {
      // Get oldest request time
      const oldest = await this.redis.zrange(windowKey, 0, 0, 'WITHSCORES');
      const resetTime = parseInt(oldest[1]) + windowMs;
      
      return {
        allowed: false,
        current: count,
        limit,
        resetTime
      };
    }
    
    // Add current request
    await this.redis.zadd(windowKey, now, `${now}-${Math.random()}`);
    await this.redis.expire(windowKey, Math.ceil(windowMs / 1000));
    
    return {
      allowed: true,
      current: count + 1,
      limit,
      remaining: limit - count - 1
    };
  }
  
  /**
   * Fixed Window Algorithm
   */
  async fixedWindow(key, limit, windowSeconds) {
    const now = Math.floor(Date.now() / 1000);
    const windowKey = `ratelimit:fixed:${key}:${Math.floor(now / windowSeconds)}`;
    
    const current = await this.redis.incr(windowKey);
    
    if (current === 1) {
      await this.redis.expire(windowKey, windowSeconds);
    }
    
    return {
      allowed: current <= limit,
      current,
      limit,
      remaining: Math.max(0, limit - current),
      resetTime: (Math.floor(now / windowSeconds) + 1) * windowSeconds
    };
  }
  
  /**
   * Adaptive Rate Limiting
   */
  async adaptiveRateLimit(key, baseLimit) {
    // Get user reputation
    const reputation = await this.getUserReputation(key);
    
    // Adjust limit based on reputation
    let adjustedLimit = baseLimit;
    
    if (reputation > 0.8) {
      adjustedLimit = baseLimit * 2; // Good user
    } else if (reputation < 0.3) {
      adjustedLimit = baseLimit * 0.5; // Suspicious user
    }
    
    return this.fixedWindow(key, adjustedLimit, 60);
  }
  
  async getUserReputation(key) {
    // Calculate based on behavior
    return 0.5; // Simplified
  }
}

/**
 * DDoS Protection Service
 */
class DDoSProtectionService {
  constructor() {
    this.redis = new Redis();
    this.suspiciousIPs = new Set();
  }
  
  /**
   * IP-based rate limiting
   */
  async checkIPRateLimit(ip) {
    const limits = [
      { window: 1, max: 10 },      // 10 requests per second
      { window: 60, max: 100 },    // 100 requests per minute
      { window: 3600, max: 1000 }  // 1000 requests per hour
    ];
    
    for (const { window, max } of limits) {
      const key = `ddos:ip:${ip}:${window}`;
      const count = await this.redis.incr(key);
      
      if (count === 1) {
        await this.redis.expire(key, window);
      }
      
      if (count > max) {
        // Mark as suspicious
        this.suspiciousIPs.add(ip);
        
        // Block temporarily
        await this.blockIP(ip, window * 2);
        
        return {
          allowed: false,
          reason: `Rate limit exceeded: ${max} requests per ${window}s`
        };
      }
    }
    
    return { allowed: true };
  }
  
  /**
   * Connection flood protection
   */
  async checkConnectionFlood(ip) {
    const key = `ddos:connections:${ip}`;
    const connections = await this.redis.incr(key);
    
    if (connections === 1) {
      await this.redis.expire(key, 10);
    }
    
    if (connections > 50) {
      await this.blockIP(ip, 300);
      return {
        allowed: false,
        reason: 'Too many concurrent connections'
      };
    }
    
    return { allowed: true };
  }
  
  /**
   * Request pattern analysis
   */
  async analyzeRequestPattern(ip, userAgent) {
    const patterns = [
      this.checkBotPattern(userAgent),
      this.checkRequestFrequency(ip),
      this.checkEndpointDistribution(ip)
    ];
    
    const results = await Promise.all(patterns);
    
    const suspicionScore = results.reduce((sum, r) => sum + r.score, 0) / results.length;
    
    if (suspicionScore > 0.7) {
      await this.blockIP(ip, 600);
      return {
        allowed: false,
        reason: 'Suspicious request pattern detected',
        score: suspicionScore
      };
    }
    
    return { allowed: true };
  }
  
  async checkBotPattern(userAgent) {
    const botPatterns = [
      /bot/i,
      /crawler/i,
      /spider/i,
      /scraper/i
    ];
    
    const isBot = botPatterns.some(pattern => pattern.test(userAgent));
    
    return { score: isBot ? 1 : 0 };
  }
  
  async checkRequestFrequency(ip) {
    // Analyze request intervals
    const key = `ddos:frequency:${ip}`;
    const timestamps = await this.redis.lrange(key, 0, -1);
    
    if (timestamps.length < 2) {
      return { score: 0 };
    }
    
    // Calculate average interval
    const intervals = [];
    for (let i = 1; i < timestamps.length; i++) {
      intervals.push(timestamps[i] - timestamps[i-1]);
    }
    
    const avgInterval = intervals.reduce((a, b) => a + b, 0) / intervals.length;
    
    // Extremely consistent intervals suggest automation
    const variance = intervals.reduce((sum, i) => sum + Math.pow(i - avgInterval, 2), 0) / intervals.length;
    
    return {
      score: variance < 100 ? 0.8 : 0.2
    };
  }
  
  async checkEndpointDistribution(ip) {
    // Check if accessing diverse endpoints or just one
    const key = `ddos:endpoints:${ip}`;
    const count = await this.redis.pfcount(key); // HyperLogLog
    
    // Low endpoint diversity suggests targeted attack
    return {
      score: count < 3 ? 0.7 : 0.2
    };
  }
  
  /**
   * Block IP temporarily
   */
  async blockIP(ip, seconds) {
    await this.redis.setex(`ddos:blocked:${ip}`, seconds, '1');
    console.log(`Blocked IP ${ip} for ${seconds} seconds`);
  }
  
  /**
   * Check if IP is blocked
   */
  async isIPBlocked(ip) {
    const blocked = await this.redis.get(`ddos:blocked:${ip}`);
    return blocked !== null;
  }
}

/**
 * Rate Limiting Middleware
 */
class RateLimitMiddleware {
  constructor() {
    this.rateLimiter = new RateLimitingService();
    this.ddosProtection = new DDoSProtectionService();
  }
  
  /**
   * General rate limit middleware
   */
  rateLimit(options = {}) {
    const {
      limit = 100,
      window = 60,
      algorithm = 'sliding',
      keyGenerator = (req) => req.ip
    } = options;
    
    return async (req, res, next) => {
      const key = keyGenerator(req);
      
      let result;
      
      if (algorithm === 'sliding') {
        result = await this.rateLimiter.slidingWindow(key, limit, window * 1000);
      } else if (algorithm === 'token') {
        result = await this.rateLimiter.tokenBucket(key, limit, limit / window);
      } else {
        result = await this.rateLimiter.fixedWindow(key, limit, window);
      }
      
      // Set rate limit headers
      res.setHeader('X-RateLimit-Limit', limit);
      res.setHeader('X-RateLimit-Remaining', result.remaining || 0);
      
      if (result.resetTime) {
        res.setHeader('X-RateLimit-Reset', result.resetTime);
      }
      
      if (!result.allowed) {
        res.setHeader('Retry-After', result.retryAfter || window);
        
        return res.status(429).json({
          error: 'Too many requests',
          retryAfter: result.retryAfter || window
        });
      }
      
      next();
    };
  }
  
  /**
   * DDoS protection middleware
   */
  ddosProtection() {
    return async (req, res, next) => {
      const ip = req.ip;
      
      // Check if IP is blocked
      if (await this.ddosProtection.isIPBlocked(ip)) {
        return res.status(403).json({
          error: 'IP temporarily blocked'
        });
      }
      
      // Check IP rate limit
      const ipCheck = await this.ddosProtection.checkIPRateLimit(ip);
      if (!ipCheck.allowed) {
        return res.status(429).json({
          error: ipCheck.reason
        });
      }
      
      // Check request pattern
      const patternCheck = await this.ddosProtection.analyzeRequestPattern(
        ip,
        req.headers['user-agent']
      );
      
      if (!patternCheck.allowed) {
        return res.status(403).json({
          error: patternCheck.reason
        });
      }
      
      next();
    };
  }
}

/**
 * ENBD Banking Application with Rate Limiting
 */
class ENBDBankingAPIWithRateLimit {
  constructor() {
    this.app = express();
    this.rateLimitMiddleware = new RateLimitMiddleware();
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    
    // DDoS protection (global)
    this.app.use(this.rateLimitMiddleware.ddosProtection());
    
    // Trust proxy (for correct IP)
    this.app.set('trust proxy', 1);
  }
  
  setupRoutes() {
    // Public API - strict rate limit
    this.app.get('/api/public/rates',
      this.rateLimitMiddleware.rateLimit({
        limit: 10,
        window: 60,
        algorithm: 'sliding'
      }),
      (req, res) => {
        res.json({ rates: [] });
      }
    );
    
    // Login - very strict rate limit
    this.app.post('/api/auth/login',
      this.rateLimitMiddleware.rateLimit({
        limit: 5,
        window: 300,
        keyGenerator: (req) => req.body.email || req.ip
      }),
      async (req, res) => {
        // Authentication logic
        res.json({ success: true });
      }
    );
    
    // Authenticated API - moderate rate limit
    this.app.get('/api/accounts',
      this.authenticate,
      this.rateLimitMiddleware.rateLimit({
        limit: 100,
        window: 60,
        keyGenerator: (req) => req.user.id
      }),
      (req, res) => {
        res.json({ accounts: [] });
      }
    );
    
    // Transaction - strict rate limit
    this.app.post('/api/transactions',
      this.authenticate,
      this.rateLimitMiddleware.rateLimit({
        limit: 10,
        window: 60,
        algorithm: 'token'
      }),
      (req, res) => {
        res.json({ success: true });
      }
    );
  }
  
  authenticate(req, res, next) {
    // JWT authentication
    next();
  }
}

module.exports = {
  RateLimitingService,
  DDoSProtectionService,
  RateLimitMiddleware,
  ENBDBankingAPIWithRateLimit
};
```

**Rate Limiting Algorithms:**

1. **Token Bucket** - Smooth rate limiting
2. **Sliding Window** - Accurate but memory-intensive
3. **Fixed Window** - Simple and fast
4. **Adaptive** - Adjusts based on user behavior

---

### Q19. How do you implement secure password storage and validation?

**Answer:**

```javascript
const bcrypt = require('bcrypt');
const argon2 = require('argon2');
const crypto = require('crypto');
const zxcvbn = require('zxcvbn');

/**
 * Password Security Service
 */
class PasswordSecurityService {
  constructor() {
    this.saltRounds = 12; // bcrypt rounds
    this.minPasswordLength = 12;
    this.passwordHistory = 5; // Remember last 5 passwords
  }
  
  /**
   * Hash password with bcrypt
   */
  async hashPassword(password) {
    return bcrypt.hash(password, this.saltRounds);
  }
  
  /**
   * Hash password with Argon2 (recommended)
   */
  async hashPasswordArgon2(password) {
    return argon2.hash(password, {
      type: argon2.argon2id,
      memoryCost: 65536,   // 64 MB
      timeCost: 3,         // 3 iterations
      parallelism: 4       // 4 threads
    });
  }
  
  /**
   * Verify password with bcrypt
   */
  async verifyPassword(password, hash) {
    return bcrypt.compare(password, hash);
  }
  
  /**
   * Verify password with Argon2
   */
  async verifyPasswordArgon2(password, hash) {
    return argon2.verify(hash, password);
  }
  
  /**
   * Validate password strength
   */
  validatePasswordStrength(password) {
    const result = zxcvbn(password);
    
    const checks = {
      minLength: password.length >= this.minPasswordLength,
      hasUppercase: /[A-Z]/.test(password),
      hasLowercase: /[a-z]/.test(password),
      hasNumber: /[0-9]/.test(password),
      hasSpecial: /[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password),
      notCommon: result.score >= 3,
      notInDictionary: !result.sequence.some(s => s.dictionary_name)
    };
    
    const allPassed = Object.values(checks).every(v => v);
    
    return {
      valid: allPassed,
      checks,
      score: result.score, // 0-4
      feedback: result.feedback,
      estimatedCrackTime: result.crack_times_display.offline_slow_hashing_1e4_per_second
    };
  }
  
  /**
   * Check password history
   */
  async checkPasswordHistory(userId, newPassword) {
    const history = await this.getPasswordHistory(userId);
    
    for (const oldHash of history) {
      const matches = await this.verifyPassword(newPassword, oldHash);
      if (matches) {
        return {
          allowed: false,
          reason: 'Password was used recently'
        };
      }
    }
    
    return { allowed: true };
  }
  
  /**
   * Generate secure random password
   */
  generateSecurePassword(length = 16) {
    const uppercase = 'ABCDEFGHJKLMNPQRSTUVWXYZ';
    const lowercase = 'abcdefghjkmnpqrstuvwxyz';
    const numbers = '23456789';
    const special = '!@#$%^&*';
    
    const all = uppercase + lowercase + numbers + special;
    
    let password = '';
    
    // Ensure at least one of each type
    password += uppercase[crypto.randomInt(uppercase.length)];
    password += lowercase[crypto.randomInt(lowercase.length)];
    password += numbers[crypto.randomInt(numbers.length)];
    password += special[crypto.randomInt(special.length)];
    
    // Fill remaining length
    for (let i = password.length; i < length; i++) {
      password += all[crypto.randomInt(all.length)];
    }
    
    // Shuffle
    return password.split('').sort(() => 0.5 - Math.random()).join('');
  }
  
  /**
   * Implement password expiration
   */
  async isPasswordExpired(userId) {
    const passwordDate = await this.getPasswordChangeDate(userId);
    const expiryDays = 90; // 90 days
    
    const daysSinceChange = (Date.now() - passwordDate) / (1000 * 60 * 60 * 24);
    
    return daysSinceChange > expiryDays;
  }
  
  /**
   * Salt password before hashing (additional layer)
   */
  async pepperPassword(password) {
    const pepper = process.env.PASSWORD_PEPPER; // Global secret
    return crypto.createHmac('sha256', pepper).update(password).digest('hex');
  }
  
  async getPasswordHistory(userId) {
    // Get from database
    return [];
  }
  
  async getPasswordChangeDate(userId) {
    return Date.now();
  }
}

/**
 * Password Reset Service
 */
class PasswordResetService {
  constructor() {
    this.redis = new Redis();
  }
  
  /**
   * Generate password reset token
   */
  async generateResetToken(userId, email) {
    const token = crypto.randomBytes(32).toString('hex');
    const hashedToken = crypto.createHash('sha256').update(token).digest('hex');
    
    // Store hashed token
    await this.redis.setex(
      `password-reset:${hashedToken}`,
      3600, // 1 hour expiry
      JSON.stringify({
        userId,
        email,
        createdAt: Date.now()
      })
    );
    
    return token; // Send unhashed token to user
  }
  
  /**
   * Verify reset token
   */
  async verifyResetToken(token) {
    const hashedToken = crypto.createHash('sha256').update(token).digest('hex');
    
    const data = await this.redis.get(`password-reset:${hashedToken}`);
    
    if (!data) {
      return {
        valid: false,
        error: 'Invalid or expired token'
      };
    }
    
    const resetData = JSON.parse(data);
    
    return {
      valid: true,
      userId: resetData.userId,
      email: resetData.email
    };
  }
  
  /**
   * Reset password
   */
  async resetPassword(token, newPassword) {
    const verification = await this.verifyResetToken(token);
    
    if (!verification.valid) {
      return verification;
    }
    
    // Validate new password
    const passwordService = new PasswordSecurityService();
    const validation = passwordService.validatePasswordStrength(newPassword);
    
    if (!validation.valid) {
      return {
        valid: false,
        error: 'Password does not meet requirements',
        checks: validation.checks
      };
    }
    
    // Check password history
    const historyCheck = await passwordService.checkPasswordHistory(
      verification.userId,
      newPassword
    );
    
    if (!historyCheck.allowed) {
      return {
        valid: false,
        error: historyCheck.reason
      };
    }
    
    // Hash new password
    const hashedPassword = await passwordService.hashPasswordArgon2(newPassword);
    
    // Update in database
    await this.updatePassword(verification.userId, hashedPassword);
    
    // Invalidate token
    const hashedToken = crypto.createHash('sha256').update(token).digest('hex');
    await this.redis.del(`password-reset:${hashedToken}`);
    
    // Revoke all active sessions
    await this.revokeAllSessions(verification.userId);
    
    return {
      valid: true,
      message: 'Password reset successful'
    };
  }
  
  async updatePassword(userId, hashedPassword) {
    // Database update
  }
  
  async revokeAllSessions(userId) {
    // Revoke sessions
  }
}

/**
 * Password Change Controller
 */
class PasswordController {
  constructor() {
    this.passwordService = new PasswordSecurityService();
    this.resetService = new PasswordResetService();
  }
  
  setupRoutes(app) {
    // Change password (authenticated)
    app.post('/api/user/change-password', this.changePassword.bind(this));
    
    // Request password reset
    app.post('/api/auth/forgot-password', this.forgotPassword.bind(this));
    
    // Reset password with token
    app.post('/api/auth/reset-password', this.resetPassword.bind(this));
    
    // Validate password strength
    app.post('/api/auth/validate-password', this.validatePassword.bind(this));
  }
  
  async changePassword(req, res) {
    const { currentPassword, newPassword } = req.body;
    const userId = req.user.id;
    
    // Get current password hash
    const user = await this.getUser(userId);
    
    // Verify current password
    const valid = await this.passwordService.verifyPasswordArgon2(
      currentPassword,
      user.passwordHash
    );
    
    if (!valid) {
      return res.status(401).json({
        error: 'Current password is incorrect'
      });
    }
    
    // Validate new password
    const validation = this.passwordService.validatePasswordStrength(newPassword);
    
    if (!validation.valid) {
      return res.status(400).json({
        error: 'Password does not meet requirements',
        checks: validation.checks,
        feedback: validation.feedback
      });
    }
    
    // Check password history
    const historyCheck = await this.passwordService.checkPasswordHistory(
      userId,
      newPassword
    );
    
    if (!historyCheck.allowed) {
      return res.status(400).json({
        error: historyCheck.reason
      });
    }
    
    // Hash and save new password
    const hashedPassword = await this.passwordService.hashPasswordArgon2(newPassword);
    await this.updatePassword(userId, hashedPassword);
    
    res.json({
      success: true,
      message: 'Password changed successfully'
    });
  }
  
  async forgotPassword(req, res) {
    const { email } = req.body;
    
    // Find user
    const user = await this.getUserByEmail(email);
    
    if (!user) {
      // Don't reveal if email exists
      return res.json({
        success: true,
        message: 'If the email exists, a reset link has been sent'
      });
    }
    
    // Generate reset token
    const token = await this.resetService.generateResetToken(user.id, email);
    
    // Send reset email
    await this.sendResetEmail(email, token);
    
    res.json({
      success: true,
      message: 'Password reset link sent to your email'
    });
  }
  
  async resetPassword(req, res) {
    const { token, newPassword } = req.body;
    
    const result = await this.resetService.resetPassword(token, newPassword);
    
    if (!result.valid) {
      return res.status(400).json({
        error: result.error,
        checks: result.checks
      });
    }
    
    res.json({
      success: true,
      message: result.message
    });
  }
  
  async validatePassword(req, res) {
    const { password } = req.body;
    
    const validation = this.passwordService.validatePasswordStrength(password);
    
    res.json({
      valid: validation.valid,
      score: validation.score,
      checks: validation.checks,
      feedback: validation.feedback,
      estimatedCrackTime: validation.estimatedCrackTime
    });
  }
  
  async getUser(userId) {
    return { id: userId, passwordHash: 'hash' };
  }
  
  async getUserByEmail(email) {
    return { id: 'user-123', email };
  }
  
  async updatePassword(userId, hashedPassword) {
    // Database update
  }
  
  async sendResetEmail(email, token) {
    // Email service
  }
}

module.exports = {
  PasswordSecurityService,
  PasswordResetService,
  PasswordController
};
```

**Password Requirements:**

- Minimum 12 characters ✅
- Uppercase + lowercase + numbers + special ✅
- Not in common password list ✅
- Not in dictionary ✅
- Not recently used ✅
- Strong entropy ✅

---

### Q20. How do you implement security headers and HTTPS?

**Answer:**

```javascript
const express = require('express');
const helmet = require('helmet');
const https = require('https');
const fs = require('fs');

/**
 * Security Headers Service
 */
class SecurityHeadersService {
  /**
   * Configure all security headers
   */
  static configureSecurityHeaders(app) {
    // Use Helmet for common headers
    app.use(helmet());
    
    // Additional custom headers
    app.use((req, res, next) => {
      // Strict Transport Security
      res.setHeader(
        'Strict-Transport-Security',
        'max-age=31536000; includeSubDomains; preload'
      );
      
      // Content Security Policy
      res.setHeader(
        'Content-Security-Policy',
        "default-src 'self'; " +
        "script-src 'self' 'nonce-{{nonce}}' https://cdn.enbd.com; " +
        "style-src 'self' 'nonce-{{nonce}}' https://cdn.enbd.com; " +
        "img-src 'self' data: https:; " +
        "font-src 'self' https://fonts.gstatic.com; " +
        "connect-src 'self' https://api.enbd.com; " +
        "frame-ancestors 'none'; " +
        "base-uri 'self'; " +
        "form-action 'self'"
      );
      
      // X-Frame-Options (prevent clickjacking)
      res.setHeader('X-Frame-Options', 'DENY');
      
      // X-Content-Type-Options (prevent MIME sniffing)
      res.setHeader('X-Content-Type-Options', 'nosniff');
      
      // X-XSS-Protection
      res.setHeader('X-XSS-Protection', '1; mode=block');
      
      // Referrer Policy
      res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
      
      // Permissions Policy (formerly Feature Policy)
      res.setHeader(
        'Permissions-Policy',
        'geolocation=(), microphone=(), camera=()'
      );
      
      // Cross-Origin Policies
      res.setHeader('Cross-Origin-Embedder-Policy', 'require-corp');
      res.setHeader('Cross-Origin-Opener-Policy', 'same-origin');
      res.setHeader('Cross-Origin-Resource-Policy', 'same-origin');
      
      // Remove fingerprinting headers
      res.removeHeader('X-Powered-By');
      
      next();
    });
  }
  
  /**
   * HSTS Preload
   */
  static hstsPreload(app) {
    app.use((req, res, next) => {
      res.setHeader(
        'Strict-Transport-Security',
        'max-age=63072000; includeSubDomains; preload'
      );
      next();
    });
  }
}

/**
 * HTTPS Configuration Service
 */
class HTTPSService {
  /**
   * Create HTTPS server with TLS 1.3
   */
  static createHTTPSServer(app) {
    const options = {
      // SSL/TLS Certificates
      key: fs.readFileSync('./certs/private-key.pem'),
      cert: fs.readFileSync('./certs/certificate.pem'),
      ca: fs.readFileSync('./certs/ca-bundle.pem'),
      
      // TLS Configuration
      minVersion: 'TLSv1.3',
      maxVersion: 'TLSv1.3',
      
      // Cipher suites (TLS 1.3)
      ciphers: [
        'TLS_AES_256_GCM_SHA384',
        'TLS_CHACHA20_POLY1305_SHA256',
        'TLS_AES_128_GCM_SHA256'
      ].join(':'),
      
      // Honor cipher order
      honorCipherOrder: true,
      
      // Disable session resumption for security
      sessionIdContext: 'enbd-banking',
      
      // OCSP Stapling
      requestOCSP: true
    };
    
    const server = https.createServer(options, app);
    
    return server;
  }
  
  /**
   * Force HTTPS redirection
   */
  static forceHTTPS() {
    return (req, res, next) => {
      if (!req.secure && req.get('x-forwarded-proto') !== 'https') {
        return res.redirect(301, `https://${req.headers.host}${req.url}`);
      }
      next();
    };
  }
  
  /**
   * Certificate pinning (for mobile apps)
   */
  static getCertificateFingerprint() {
    const cert = fs.readFileSync('./certs/certificate.pem');
    const fingerprint = crypto
      .createHash('sha256')
      .update(cert)
      .digest('base64');
    
    return {
      'pin-sha256': fingerprint,
      'max-age': 5184000,
      includeSubDomains: true
    };
  }
}

/**
 * Complete Secure ENBD Banking Application
 */
class SecureENBDApplication {
  constructor() {
    this.app = express();
    this.setupSecurity();
    this.setupRoutes();
  }
  
  setupSecurity() {
    // Force HTTPS
    this.app.use(HTTPSService.forceHTTPS());
    
    // Security headers
    SecurityHeadersService.configureSecurityHeaders(this.app);
    
    // HSTS Preload
    SecurityHeadersService.hstsPreload(this.app);
    
    // Rate limiting
    this.app.use(this.getRateLimiter());
    
    // Body parsers with limits
    this.app.use(express.json({ limit: '10kb' }));
    this.app.use(express.urlencoded({ extended: true, limit: '10kb' }));
  }
  
  setupRoutes() {
    this.app.get('/', (req, res) => {
      res.json({
        message: 'ENBD Banking API',
        security: 'All security measures active'
      });
    });
    
    // Security.txt (RFC 9116)
    this.app.get('/.well-known/security.txt', (req, res) => {
      res.type('text/plain');
      res.send(`
Contact: mailto:security@enbd.com
Expires: 2025-12-31T23:59:59.000Z
Encryption: https://enbd.com/pgp-key.txt
Preferred-Languages: en, ar
Canonical: https://enbd.com/.well-known/security.txt
Policy: https://enbd.com/security-policy
      `.trim());
    });
  }
  
  getRateLimiter() {
    const rateLimit = require('express-rate-limit');
    
    return rateLimit({
      windowMs: 15 * 60 * 1000,
      max: 100,
      message: 'Too many requests'
    });
  }
  
  start(port = 443) {
    const server = HTTPSService.createHTTPSServer(this.app);
    
    server.listen(port, () => {
      console.log(`🔒 Secure ENBD API running on https://localhost:${port}`);
      console.log('✅ TLS 1.3 enabled');
      console.log('✅ All security headers configured');
      console.log('✅ HSTS preload ready');
    });
  }
}

module.exports = {
  SecurityHeadersService,
  HTTPSService,
  SecureENBDApplication
};
```

**Security Headers Checklist:**

1. ✅ **HSTS** - Force HTTPS
2. ✅ **CSP** - Content Security Policy
3. ✅ **X-Frame-Options** - Clickjacking protection
4. ✅ **X-Content-Type-Options** - MIME sniffing protection
5. ✅ **X-XSS-Protection** - XSS filter
6. ✅ **Referrer-Policy** - Control referrer information
7. ✅ **Permissions-Policy** - Control browser features
8. ✅ **CORS** - Cross-origin resource sharing

---

**Summary Q16-Q20:**
- XSS Prevention with sanitization & CSP ✅
- CSRF Protection (multiple strategies) ✅
- Rate Limiting & DDoS Protection ✅
- Secure Password Storage (Argon2/bcrypt) ✅
- Security Headers & HTTPS Configuration ✅

Continuing with remaining security questions...
