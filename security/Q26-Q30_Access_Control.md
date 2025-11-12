# Security Questions 26-30: Access Control & Authorization

---

### Q26. How do you implement Role-Based Access Control (RBAC)?

**Answer:**

```javascript
const express = require('express');

/**
 * RBAC Service
 */
class RBACService {
  constructor() {
    this.roles = new Map();
    this.permissions = new Map();
    this.userRoles = new Map();
    
    this.initializeRoles();
  }
  
  /**
   * Initialize role hierarchy
   */
  initializeRoles() {
    // Define permissions
    this.definePermission('accounts:read', 'Read account information');
    this.definePermission('accounts:write', 'Modify account information');
    this.definePermission('accounts:delete', 'Delete accounts');
    
    this.definePermission('transactions:read', 'Read transactions');
    this.definePermission('transactions:create', 'Create transactions');
    this.definePermission('transactions:approve', 'Approve transactions');
    
    this.definePermission('users:read', 'Read user data');
    this.definePermission('users:write', 'Modify users');
    this.definePermission('users:delete', 'Delete users');
    
    this.definePermission('admin:system', 'System administration');
    
    // Define roles with permissions
    this.defineRole('customer', [
      'accounts:read',
      'transactions:read',
      'transactions:create'
    ]);
    
    this.defineRole('teller', [
      'accounts:read',
      'accounts:write',
      'transactions:read',
      'transactions:create',
      'users:read'
    ]);
    
    this.defineRole('manager', [
      'accounts:read',
      'accounts:write',
      'transactions:read',
      'transactions:create',
      'transactions:approve',
      'users:read',
      'users:write'
    ]);
    
    this.defineRole('admin', [
      'accounts:read',
      'accounts:write',
      'accounts:delete',
      'transactions:read',
      'transactions:create',
      'transactions:approve',
      'users:read',
      'users:write',
      'users:delete',
      'admin:system'
    ]);
  }
  
  /**
   * Define permission
   */
  definePermission(name, description) {
    this.permissions.set(name, { name, description });
  }
  
  /**
   * Define role with permissions
   */
  defineRole(roleName, permissions) {
    this.roles.set(roleName, {
      name: roleName,
      permissions: new Set(permissions)
    });
  }
  
  /**
   * Assign role to user
   */
  async assignRole(userId, roleName) {
    if (!this.roles.has(roleName)) {
      throw new Error(`Role ${roleName} does not exist`);
    }
    
    if (!this.userRoles.has(userId)) {
      this.userRoles.set(userId, new Set());
    }
    
    this.userRoles.get(userId).add(roleName);
    
    // Log role assignment
    await this.logRoleChange(userId, 'ASSIGNED', roleName);
  }
  
  /**
   * Remove role from user
   */
  async removeRole(userId, roleName) {
    if (this.userRoles.has(userId)) {
      this.userRoles.get(userId).delete(roleName);
      await this.logRoleChange(userId, 'REMOVED', roleName);
    }
  }
  
  /**
   * Get user roles
   */
  getUserRoles(userId) {
    return Array.from(this.userRoles.get(userId) || []);
  }
  
  /**
   * Get all permissions for user
   */
  getUserPermissions(userId) {
    const userRoles = this.getUserRoles(userId);
    const permissions = new Set();
    
    for (const roleName of userRoles) {
      const role = this.roles.get(roleName);
      if (role) {
        role.permissions.forEach(perm => permissions.add(perm));
      }
    }
    
    return Array.from(permissions);
  }
  
  /**
   * Check if user has permission
   */
  hasPermission(userId, permission) {
    const permissions = this.getUserPermissions(userId);
    return permissions.includes(permission);
  }
  
  /**
   * Check if user has any of the permissions
   */
  hasAnyPermission(userId, permissions) {
    return permissions.some(perm => this.hasPermission(userId, perm));
  }
  
  /**
   * Check if user has all permissions
   */
  hasAllPermissions(userId, permissions) {
    return permissions.every(perm => this.hasPermission(userId, perm));
  }
  
  /**
   * Check if user has role
   */
  hasRole(userId, roleName) {
    const roles = this.getUserRoles(userId);
    return roles.includes(roleName);
  }
  
  async logRoleChange(userId, action, roleName) {
    console.log(`Role ${action}: User ${userId} - ${roleName}`);
  }
}

/**
 * RBAC Middleware
 */
class RBACMiddleware {
  constructor(rbacService) {
    this.rbacService = rbacService;
  }
  
  /**
   * Require specific permission
   */
  requirePermission(permission) {
    return async (req, res, next) => {
      const userId = req.user?.id;
      
      if (!userId) {
        return res.status(401).json({
          error: 'Authentication required'
        });
      }
      
      if (!this.rbacService.hasPermission(userId, permission)) {
        return res.status(403).json({
          error: 'Insufficient permissions',
          required: permission
        });
      }
      
      next();
    };
  }
  
  /**
   * Require any of the permissions
   */
  requireAnyPermission(...permissions) {
    return async (req, res, next) => {
      const userId = req.user?.id;
      
      if (!userId) {
        return res.status(401).json({
          error: 'Authentication required'
        });
      }
      
      if (!this.rbacService.hasAnyPermission(userId, permissions)) {
        return res.status(403).json({
          error: 'Insufficient permissions',
          required: permissions
        });
      }
      
      next();
    };
  }
  
  /**
   * Require specific role
   */
  requireRole(roleName) {
    return async (req, res, next) => {
      const userId = req.user?.id;
      
      if (!userId) {
        return res.status(401).json({
          error: 'Authentication required'
        });
      }
      
      if (!this.rbacService.hasRole(userId, roleName)) {
        return res.status(403).json({
          error: 'Insufficient role',
          required: roleName
        });
      }
      
      next();
    };
  }
  
  /**
   * Require resource ownership or permission
   */
  requireOwnershipOrPermission(resourceGetter, permission) {
    return async (req, res, next) => {
      const userId = req.user?.id;
      const resourceOwnerId = await resourceGetter(req);
      
      // Check ownership
      if (userId === resourceOwnerId) {
        return next();
      }
      
      // Check permission
      if (this.rbacService.hasPermission(userId, permission)) {
        return next();
      }
      
      res.status(403).json({
        error: 'Access denied'
      });
    };
  }
}

/**
 * ENBD Banking RBAC Application
 */
class ENBDBankingRBAC {
  constructor() {
    this.app = express();
    this.rbacService = new RBACService();
    this.rbacMiddleware = new RBACMiddleware(this.rbacService);
    
    this.setupRoutes();
  }
  
  setupRoutes() {
    // Public route
    this.app.get('/api/health', (req, res) => {
      res.json({ status: 'ok' });
    });
    
    // Customer can read their own accounts
    this.app.get('/api/accounts',
      this.authenticate.bind(this),
      this.rbacMiddleware.requirePermission('accounts:read'),
      async (req, res) => {
        const accounts = await this.getAccounts(req.user.id);
        res.json({ accounts });
      }
    );
    
    // Teller can create transactions
    this.app.post('/api/transactions',
      this.authenticate.bind(this),
      this.rbacMiddleware.requirePermission('transactions:create'),
      async (req, res) => {
        const transaction = await this.createTransaction(req.body);
        res.json({ transaction });
      }
    );
    
    // Manager can approve transactions
    this.app.post('/api/transactions/:id/approve',
      this.authenticate.bind(this),
      this.rbacMiddleware.requirePermission('transactions:approve'),
      async (req, res) => {
        await this.approveTransaction(req.params.id);
        res.json({ success: true });
      }
    );
    
    // Admin can manage users
    this.app.post('/api/users',
      this.authenticate.bind(this),
      this.rbacMiddleware.requireRole('admin'),
      async (req, res) => {
        const user = await this.createUser(req.body);
        res.json({ user });
      }
    );
    
    // Admin can assign roles
    this.app.post('/api/users/:id/roles',
      this.authenticate.bind(this),
      this.rbacMiddleware.requirePermission('admin:system'),
      async (req, res) => {
        await this.rbacService.assignRole(req.params.id, req.body.role);
        res.json({ success: true });
      }
    );
  }
  
  authenticate(req, res, next) {
    // JWT authentication
    req.user = { id: 'user-123' };
    next();
  }
  
  async getAccounts(userId) {
    return [];
  }
  
  async createTransaction(data) {
    return {};
  }
  
  async approveTransaction(id) {
    // Approve logic
  }
  
  async createUser(data) {
    return {};
  }
}

module.exports = {
  RBACService,
  RBACMiddleware,
  ENBDBankingRBAC
};
```

**RBAC Best Practices:**

1. ✅ **Principle of Least Privilege**
2. ✅ **Role hierarchy**
3. ✅ **Permission-based authorization**
4. ✅ **Audit role changes**
5. ✅ **Resource ownership checks**

---

### Q27. How do you implement Attribute-Based Access Control (ABAC)?

**Answer:**

```javascript
/**
 * ABAC Policy Engine
 */
class ABACPolicyEngine {
  constructor() {
    this.policies = [];
  }
  
  /**
   * Define access policy
   */
  definePolicy(policy) {
    this.policies.push(policy);
  }
  
  /**
   * Evaluate access request
   */
  async evaluate(subject, action, resource, context = {}) {
    // Collect all attributes
    const attributes = {
      subject: await this.getSubjectAttributes(subject),
      resource: await this.getResourceAttributes(resource),
      action,
      context: await this.getContextAttributes(context)
    };
    
    // Evaluate all policies
    const results = this.policies.map(policy => 
      this.evaluatePolicy(policy, attributes)
    );
    
    // Determine final decision (deny overrides)
    const hasDeny = results.some(r => r.effect === 'deny');
    const hasAllow = results.some(r => r.effect === 'allow');
    
    if (hasDeny) {
      return {
        decision: 'DENY',
        reason: 'Explicit deny policy matched'
      };
    }
    
    if (hasAllow) {
      return {
        decision: 'ALLOW',
        reason: 'Allow policy matched'
      };
    }
    
    return {
      decision: 'DENY',
      reason: 'No matching allow policy'
    };
  }
  
  /**
   * Evaluate single policy
   */
  evaluatePolicy(policy, attributes) {
    // Check if policy applies
    const matches = policy.conditions.every(condition => 
      this.evaluateCondition(condition, attributes)
    );
    
    return {
      effect: matches ? policy.effect : 'no-match',
      policy: policy.name
    };
  }
  
  /**
   * Evaluate condition
   */
  evaluateCondition(condition, attributes) {
    const { attribute, operator, value } = condition;
    const actualValue = this.getAttributeValue(attribute, attributes);
    
    switch (operator) {
      case 'equals':
        return actualValue === value;
      
      case 'notEquals':
        return actualValue !== value;
      
      case 'in':
        return Array.isArray(value) && value.includes(actualValue);
      
      case 'notIn':
        return Array.isArray(value) && !value.includes(actualValue);
      
      case 'greaterThan':
        return actualValue > value;
      
      case 'lessThan':
        return actualValue < value;
      
      case 'contains':
        return Array.isArray(actualValue) && actualValue.includes(value);
      
      case 'startsWith':
        return String(actualValue).startsWith(value);
      
      case 'matches':
        return new RegExp(value).test(actualValue);
      
      default:
        return false;
    }
  }
  
  /**
   * Get attribute value from nested path
   */
  getAttributeValue(path, attributes) {
    const parts = path.split('.');
    let value = attributes;
    
    for (const part of parts) {
      value = value?.[part];
    }
    
    return value;
  }
  
  /**
   * Get subject attributes
   */
  async getSubjectAttributes(subject) {
    return {
      id: subject.id,
      role: subject.role,
      department: subject.department,
      clearanceLevel: subject.clearanceLevel,
      location: subject.location,
      employeeType: subject.employeeType
    };
  }
  
  /**
   * Get resource attributes
   */
  async getResourceAttributes(resource) {
    return {
      id: resource.id,
      type: resource.type,
      owner: resource.owner,
      classification: resource.classification,
      department: resource.department,
      tags: resource.tags
    };
  }
  
  /**
   * Get context attributes
   */
  async getContextAttributes(context) {
    return {
      time: new Date(),
      dayOfWeek: new Date().getDay(),
      hour: new Date().getHours(),
      ip: context.ip,
      location: context.location,
      device: context.device,
      riskScore: context.riskScore || 0
    };
  }
}

/**
 * ENBD ABAC Policies
 */
class ENBDABACPolicies {
  static initialize(engine) {
    // Policy 1: Regular employees can read accounts during business hours
    engine.definePolicy({
      name: 'BusinessHoursAccountAccess',
      effect: 'allow',
      conditions: [
        { attribute: 'subject.employeeType', operator: 'equals', value: 'regular' },
        { attribute: 'action', operator: 'equals', value: 'read' },
        { attribute: 'resource.type', operator: 'equals', value: 'account' },
        { attribute: 'context.hour', operator: 'greaterThan', value: 8 },
        { attribute: 'context.hour', operator: 'lessThan', value: 18 }
      ]
    });
    
    // Policy 2: Managers can approve transactions up to 100,000 AED
    engine.definePolicy({
      name: 'ManagerTransactionApproval',
      effect: 'allow',
      conditions: [
        { attribute: 'subject.role', operator: 'equals', value: 'manager' },
        { attribute: 'action', operator: 'equals', value: 'approve' },
        { attribute: 'resource.type', operator: 'equals', value: 'transaction' },
        { attribute: 'resource.amount', operator: 'lessThan', value: 100000 }
      ]
    });
    
    // Policy 3: Same department access
    engine.definePolicy({
      name: 'SameDepartmentAccess',
      effect: 'allow',
      conditions: [
        { attribute: 'subject.department', operator: 'equals', value: '{{resource.department}}' },
        { attribute: 'action', operator: 'in', value: ['read', 'update'] }
      ]
    });
    
    // Policy 4: High clearance for sensitive data
    engine.definePolicy({
      name: 'SensitiveDataAccess',
      effect: 'allow',
      conditions: [
        { attribute: 'subject.clearanceLevel', operator: 'greaterThan', value: 3 },
        { attribute: 'resource.classification', operator: 'equals', value: 'sensitive' }
      ]
    });
    
    // Policy 5: Deny access from high-risk locations
    engine.definePolicy({
      name: 'DenyHighRiskLocation',
      effect: 'deny',
      conditions: [
        { attribute: 'context.riskScore', operator: 'greaterThan', value: 7 }
      ]
    });
    
    // Policy 6: Deny access outside UAE for certain operations
    engine.definePolicy({
      name: 'GeoRestriction',
      effect: 'deny',
      conditions: [
        { attribute: 'context.location', operator: 'notEquals', value: 'UAE' },
        { attribute: 'action', operator: 'in', value: ['transfer', 'withdraw'] }
      ]
    });
    
    // Policy 7: Owner can always access their resources
    engine.definePolicy({
      name: 'OwnerFullAccess',
      effect: 'allow',
      conditions: [
        { attribute: 'subject.id', operator: 'equals', value: '{{resource.owner}}' }
      ]
    });
  }
}

/**
 * ABAC Middleware
 */
class ABACMiddleware {
  constructor(policyEngine) {
    this.policyEngine = policyEngine;
  }
  
  /**
   * Enforce ABAC policy
   */
  enforce() {
    return async (req, res, next) => {
      const subject = req.user;
      const action = this.getAction(req);
      const resource = await this.getResource(req);
      const context = this.getContext(req);
      
      const decision = await this.policyEngine.evaluate(
        subject,
        action,
        resource,
        context
      );
      
      if (decision.decision === 'DENY') {
        return res.status(403).json({
          error: 'Access denied',
          reason: decision.reason
        });
      }
      
      next();
    };
  }
  
  getAction(req) {
    const methodToAction = {
      'GET': 'read',
      'POST': 'create',
      'PUT': 'update',
      'PATCH': 'update',
      'DELETE': 'delete'
    };
    
    return methodToAction[req.method] || 'read';
  }
  
  async getResource(req) {
    // Load resource from database
    return {
      id: req.params.id,
      type: 'account',
      owner: 'user-123',
      classification: 'sensitive',
      department: 'retail-banking'
    };
  }
  
  getContext(req) {
    return {
      ip: req.ip,
      location: req.headers['x-location'] || 'UAE',
      device: req.headers['user-agent'],
      riskScore: 0
    };
  }
}

/**
 * Dynamic Policy Builder
 */
class DynamicPolicyBuilder {
  /**
   * Build policy from JSON
   */
  static buildPolicy(json) {
    return {
      name: json.name,
      effect: json.effect,
      conditions: json.conditions.map(c => ({
        attribute: c.attribute,
        operator: c.operator,
        value: c.value
      }))
    };
  }
  
  /**
   * Policy template for common scenarios
   */
  static createTimeBasedPolicy(name, startHour, endHour) {
    return {
      name,
      effect: 'allow',
      conditions: [
        { attribute: 'context.hour', operator: 'greaterThan', value: startHour },
        { attribute: 'context.hour', operator: 'lessThan', value: endHour }
      ]
    };
  }
  
  static createAmountLimitPolicy(name, role, maxAmount) {
    return {
      name,
      effect: 'allow',
      conditions: [
        { attribute: 'subject.role', operator: 'equals', value: role },
        { attribute: 'resource.amount', operator: 'lessThan', value: maxAmount }
      ]
    };
  }
}

/**
 * ABAC with Machine Learning Risk Score
 */
class MLRiskScoringABAC {
  /**
   * Calculate risk score based on context
   */
  async calculateRiskScore(subject, resource, context) {
    let score = 0;
    
    // Unusual access time
    if (context.hour < 6 || context.hour > 22) {
      score += 2;
    }
    
    // Foreign location
    if (context.location !== subject.location) {
      score += 3;
    }
    
    // High-value transaction
    if (resource.amount > 50000) {
      score += 2;
    }
    
    // New device
    if (!await this.isKnownDevice(subject.id, context.device)) {
      score += 3;
    }
    
    return score;
  }
  
  async isKnownDevice(userId, device) {
    return false; // Simplified
  }
}

module.exports = {
  ABACPolicyEngine,
  ENBDABACPolicies,
  ABACMiddleware,
  DynamicPolicyBuilder,
  MLRiskScoringABAC
};
```

**ABAC Advantages:**

1. ✅ **Fine-grained control**
2. ✅ **Context-aware decisions**
3. ✅ **Dynamic policies**
4. ✅ **Flexible conditions**
5. ✅ **Time-based access**
6. ✅ **Location-based access**

---

### Q28. How do you implement secure session management?

**Answer:**

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const Redis = require('ioredis');
const crypto = require('crypto');

/**
 * Secure Session Management Service
 */
class SecureSessionService {
  constructor() {
    this.redis = new Redis({
      host: 'localhost',
      port: 6379,
      password: process.env.REDIS_PASSWORD,
      tls: process.env.NODE_ENV === 'production' ? {} : undefined
    });
  }
  
  /**
   * Create session middleware
   */
  createSessionMiddleware() {
    return session({
      store: new RedisStore({
        client: this.redis,
        prefix: 'enbd:sess:',
        ttl: 3600 // 1 hour
      }),
      
      secret: process.env.SESSION_SECRET,
      
      name: 'enbd.sid', // Custom name (don't use default 'connect.sid')
      
      resave: false,
      saveUninitialized: false,
      
      cookie: {
        secure: true,           // HTTPS only
        httpOnly: true,         // Not accessible via JavaScript
        sameSite: 'strict',     // CSRF protection
        maxAge: 3600000,        // 1 hour
        domain: '.enbd.com'
      },
      
      // Generate secure session ID
      genid: () => {
        return crypto.randomBytes(32).toString('hex');
      }
    });
  }
  
  /**
   * Create session for user
   */
  async createSession(userId, metadata = {}) {
    const sessionId = crypto.randomBytes(32).toString('hex');
    
    const sessionData = {
      userId,
      createdAt: Date.now(),
      lastActivity: Date.now(),
      ip: metadata.ip,
      userAgent: metadata.userAgent,
      deviceId: metadata.deviceId,
      loginMethod: metadata.loginMethod
    };
    
    await this.redis.setex(
      `enbd:sess:${sessionId}`,
      3600, // 1 hour
      JSON.stringify(sessionData)
    );
    
    return sessionId;
  }
  
  /**
   * Get session data
   */
  async getSession(sessionId) {
    const data = await this.redis.get(`enbd:sess:${sessionId}`);
    
    if (!data) {
      return null;
    }
    
    return JSON.parse(data);
  }
  
  /**
   * Update session activity
   */
  async updateActivity(sessionId) {
    const session = await this.getSession(sessionId);
    
    if (!session) {
      return false;
    }
    
    session.lastActivity = Date.now();
    
    await this.redis.setex(
      `enbd:sess:${sessionId}`,
      3600,
      JSON.stringify(session)
    );
    
    return true;
  }
  
  /**
   * Destroy session
   */
  async destroySession(sessionId) {
    await this.redis.del(`enbd:sess:${sessionId}`);
  }
  
  /**
   * Destroy all user sessions (logout from all devices)
   */
  async destroyAllUserSessions(userId) {
    const pattern = 'enbd:sess:*';
    const keys = await this.redis.keys(pattern);
    
    for (const key of keys) {
      const data = await this.redis.get(key);
      const session = JSON.parse(data);
      
      if (session.userId === userId) {
        await this.redis.del(key);
      }
    }
  }
  
  /**
   * Get all user sessions
   */
  async getUserSessions(userId) {
    const pattern = 'enbd:sess:*';
    const keys = await this.redis.keys(pattern);
    const sessions = [];
    
    for (const key of keys) {
      const data = await this.redis.get(key);
      const session = JSON.parse(data);
      
      if (session.userId === userId) {
        sessions.push({
          sessionId: key.replace('enbd:sess:', ''),
          ...session
        });
      }
    }
    
    return sessions;
  }
  
  /**
   * Session regeneration (after login/privilege escalation)
   */
  async regenerateSession(oldSessionId, userId, metadata) {
    // Destroy old session
    await this.destroySession(oldSessionId);
    
    // Create new session
    return await this.createSession(userId, metadata);
  }
  
  /**
   * Implement session timeout
   */
  async checkTimeout(sessionId, timeoutMs = 1800000) {
    const session = await this.getSession(sessionId);
    
    if (!session) {
      return { valid: false, reason: 'Session not found' };
    }
    
    const inactiveTime = Date.now() - session.lastActivity;
    
    if (inactiveTime > timeoutMs) {
      await this.destroySession(sessionId);
      return {
        valid: false,
        reason: 'Session timeout due to inactivity'
      };
    }
    
    return { valid: true };
  }
  
  /**
   * Detect session hijacking
   */
  async detectHijacking(sessionId, currentIp, currentUserAgent) {
    const session = await this.getSession(sessionId);
    
    if (!session) {
      return { hijacked: false };
    }
    
    // Check IP change
    if (session.ip !== currentIp) {
      await this.destroySession(sessionId);
      return {
        hijacked: true,
        reason: 'IP address changed'
      };
    }
    
    // Check User-Agent change
    if (session.userAgent !== currentUserAgent) {
      await this.destroySession(sessionId);
      return {
        hijacked: true,
        reason: 'User agent changed'
      };
    }
    
    return { hijacked: false };
  }
}

/**
 * Session Security Middleware
 */
class SessionSecurityMiddleware {
  constructor(sessionService) {
    this.sessionService = sessionService;
  }
  
  /**
   * Validate session middleware
   */
  validateSession() {
    return async (req, res, next) => {
      const sessionId = req.session.id;
      
      if (!sessionId) {
        return res.status(401).json({
          error: 'No session'
        });
      }
      
      // Check timeout
      const timeoutCheck = await this.sessionService.checkTimeout(sessionId);
      
      if (!timeoutCheck.valid) {
        return res.status(401).json({
          error: timeoutCheck.reason
        });
      }
      
      // Check hijacking
      const hijackCheck = await this.sessionService.detectHijacking(
        sessionId,
        req.ip,
        req.headers['user-agent']
      );
      
      if (hijackCheck.hijacked) {
        return res.status(401).json({
          error: 'Session invalid',
          reason: hijackCheck.reason
        });
      }
      
      // Update activity
      await this.sessionService.updateActivity(sessionId);
      
      next();
    };
  }
  
  /**
   * Require fresh authentication for sensitive operations
   */
  requireFreshAuth(maxAge = 300000) {
    return async (req, res, next) => {
      const session = await this.sessionService.getSession(req.session.id);
      
      if (!session) {
        return res.status(401).json({
          error: 'Session not found'
        });
      }
      
      const age = Date.now() - session.createdAt;
      
      if (age > maxAge) {
        return res.status(403).json({
          error: 'Reauthentication required',
          reason: 'Session too old for sensitive operation'
        });
      }
      
      next();
    };
  }
  
  /**
   * Implement concurrent session limit
   */
  limitConcurrentSessions(maxSessions = 3) {
    return async (req, res, next) => {
      const userId = req.user.id;
      const sessions = await this.sessionService.getUserSessions(userId);
      
      if (sessions.length >= maxSessions) {
        // Remove oldest session
        const oldest = sessions.sort((a, b) => a.createdAt - b.createdAt)[0];
        await this.sessionService.destroySession(oldest.sessionId);
      }
      
      next();
    };
  }
}

/**
 * Secure Cookie Configuration
 */
class SecureCookieService {
  /**
   * Set secure cookie
   */
  static setSecureCookie(res, name, value, options = {}) {
    res.cookie(name, value, {
      secure: true,
      httpOnly: true,
      sameSite: 'strict',
      maxAge: options.maxAge || 3600000,
      signed: true,
      domain: options.domain || '.enbd.com',
      path: options.path || '/'
    });
  }
  
  /**
   * Clear cookie
   */
  static clearCookie(res, name) {
    res.clearCookie(name, {
      secure: true,
      httpOnly: true,
      sameSite: 'strict',
      domain: '.enbd.com'
    });
  }
}

/**
 * Device Fingerprinting for Session Security
 */
class DeviceFingerprintService {
  /**
   * Generate device fingerprint
   */
  generateFingerprint(req) {
    const components = [
      req.headers['user-agent'],
      req.headers['accept-language'],
      req.headers['accept-encoding'],
      this.getScreenResolution(req),
      this.getTimezone(req)
    ];
    
    return crypto
      .createHash('sha256')
      .update(components.join('|'))
      .digest('hex');
  }
  
  getScreenResolution(req) {
    return req.headers['x-screen-resolution'] || 'unknown';
  }
  
  getTimezone(req) {
    return req.headers['x-timezone'] || 'unknown';
  }
}

/**
 * ENBD Banking Session Management
 */
class ENBDBankingSessionApp {
  constructor() {
    this.app = express();
    this.sessionService = new SecureSessionService();
    this.sessionMiddleware = new SessionSecurityMiddleware(this.sessionService);
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    this.app.use(express.json());
    this.app.use(this.sessionService.createSessionMiddleware());
  }
  
  setupRoutes() {
    // Login
    this.app.post('/api/auth/login', async (req, res) => {
      const { email, password } = req.body;
      
      // Authenticate user
      const user = await this.authenticateUser(email, password);
      
      if (!user) {
        return res.status(401).json({ error: 'Invalid credentials' });
      }
      
      // Create session
      const sessionId = await this.sessionService.createSession(user.id, {
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        loginMethod: 'password'
      });
      
      res.json({
        success: true,
        sessionId
      });
    });
    
    // Protected route
    this.app.get('/api/accounts',
      this.sessionMiddleware.validateSession(),
      async (req, res) => {
        const accounts = await this.getAccounts(req.user.id);
        res.json({ accounts });
      }
    );
    
    // Sensitive operation (requires fresh auth)
    this.app.post('/api/transfer',
      this.sessionMiddleware.validateSession(),
      this.sessionMiddleware.requireFreshAuth(300000), // 5 minutes
      async (req, res) => {
        await this.processTransfer(req.body);
        res.json({ success: true });
      }
    );
    
    // Logout
    this.app.post('/api/auth/logout', async (req, res) => {
      await this.sessionService.destroySession(req.session.id);
      res.json({ success: true });
    });
    
    // Logout from all devices
    this.app.post('/api/auth/logout-all', async (req, res) => {
      await this.sessionService.destroyAllUserSessions(req.user.id);
      res.json({ success: true });
    });
    
    // List active sessions
    this.app.get('/api/sessions', async (req, res) => {
      const sessions = await this.sessionService.getUserSessions(req.user.id);
      res.json({ sessions });
    });
  }
  
  async authenticateUser(email, password) {
    return { id: 'user-123', email };
  }
  
  async getAccounts(userId) {
    return [];
  }
  
  async processTransfer(data) {
    // Transfer logic
  }
}

module.exports = {
  SecureSessionService,
  SessionSecurityMiddleware,
  SecureCookieService,
  DeviceFingerprintService,
  ENBDBankingSessionApp
};
```

**Session Security Best Practices:**

1. ✅ **HTTPS only cookies**
2. ✅ **HttpOnly flag** (prevent XSS)
3. ✅ **SameSite=Strict** (prevent CSRF)
4. ✅ **Session regeneration** after login
5. ✅ **Activity timeout** (30 minutes)
6. ✅ **Absolute timeout** (1 hour)
7. ✅ **Hijacking detection** (IP/UA change)
8. ✅ **Concurrent session limits**

---

### Q29. How do you implement API security (API keys, OAuth scopes)?

**Answer:**

```javascript
const crypto = require('crypto');

/**
 * API Key Management Service
 */
class APIKeyManagementService {
  constructor() {
    this.redis = new Redis();
  }
  
  /**
   * Generate API key
   */
  async generateAPIKey(userId, name, scopes = [], expiresIn = null) {
    // Generate key: enbd_live_<random>
    const prefix = process.env.NODE_ENV === 'production' ? 'enbd_live' : 'enbd_test';
    const random = crypto.randomBytes(24).toString('base64url');
    const apiKey = `${prefix}_${random}`;
    
    // Hash key for storage
    const hashedKey = crypto.createHash('sha256').update(apiKey).digest('hex');
    
    // Store metadata
    const metadata = {
      userId,
      name,
      scopes,
      hashedKey,
      createdAt: Date.now(),
      expiresAt: expiresIn ? Date.now() + expiresIn : null,
      lastUsed: null,
      usageCount: 0
    };
    
    await this.storeAPIKey(hashedKey, metadata);
    
    // Return unhashed key ONCE (never stored)
    return {
      apiKey, // Show only once
      keyId: hashedKey.substring(0, 12),
      scopes,
      expiresAt: metadata.expiresAt
    };
  }
  
  /**
   * Validate API key
   */
  async validateAPIKey(apiKey) {
    const hashedKey = crypto.createHash('sha256').update(apiKey).digest('hex');
    
    const metadata = await this.getAPIKey(hashedKey);
    
    if (!metadata) {
      return {
        valid: false,
        error: 'Invalid API key'
      };
    }
    
    // Check expiration
    if (metadata.expiresAt && Date.now() > metadata.expiresAt) {
      return {
        valid: false,
        error: 'API key expired'
      };
    }
    
    // Update usage
    await this.updateUsage(hashedKey);
    
    return {
      valid: true,
      userId: metadata.userId,
      scopes: metadata.scopes
    };
  }
  
  /**
   * Revoke API key
   */
  async revokeAPIKey(keyId) {
    await this.redis.del(`apikey:${keyId}`);
  }
  
  /**
   * List user API keys
   */
  async listUserAPIKeys(userId) {
    const keys = await this.redis.keys('apikey:*');
    const userKeys = [];
    
    for (const key of keys) {
      const data = await this.redis.get(key);
      const metadata = JSON.parse(data);
      
      if (metadata.userId === userId) {
        userKeys.push({
          keyId: key.replace('apikey:', '').substring(0, 12),
          name: metadata.name,
          scopes: metadata.scopes,
          createdAt: metadata.createdAt,
          lastUsed: metadata.lastUsed,
          usageCount: metadata.usageCount
        });
      }
    }
    
    return userKeys;
  }
  
  /**
   * Rotate API key
   */
  async rotateAPIKey(oldKeyId, userId) {
    const oldMetadata = await this.getAPIKey(oldKeyId);
    
    if (!oldMetadata || oldMetadata.userId !== userId) {
      throw new Error('Invalid API key');
    }
    
    // Generate new key with same scopes
    const newKey = await this.generateAPIKey(
      userId,
      oldMetadata.name,
      oldMetadata.scopes
    );
    
    // Revoke old key
    await this.revokeAPIKey(oldKeyId);
    
    return newKey;
  }
  
  async storeAPIKey(hashedKey, metadata) {
    await this.redis.set(`apikey:${hashedKey}`, JSON.stringify(metadata));
  }
  
  async getAPIKey(hashedKey) {
    const data = await this.redis.get(`apikey:${hashedKey}`);
    return data ? JSON.parse(data) : null;
  }
  
  async updateUsage(hashedKey) {
    const metadata = await this.getAPIKey(hashedKey);
    metadata.lastUsed = Date.now();
    metadata.usageCount++;
    await this.storeAPIKey(hashedKey, metadata);
  }
}

/**
 * OAuth Scope Management
 */
class OAuthScopeService {
  constructor() {
    this.scopes = new Map();
    this.definedScopes();
  }
  
  /**
   * Define available scopes
   */
  definedScopes() {
    this.defineScope('accounts:read', 'Read account information');
    this.defineScope('accounts:write', 'Modify accounts');
    
    this.defineScope('transactions:read', 'Read transaction history');
    this.defineScope('transactions:create', 'Create transactions');
    
    this.defineScope('cards:read', 'Read card information');
    this.defineScope('cards:manage', 'Manage cards');
    
    this.defineScope('profile:read', 'Read user profile');
    this.defineScope('profile:write', 'Update user profile');
  }
  
  defineScope(name, description) {
    this.scopes.set(name, { name, description });
  }
  
  /**
   * Validate scopes
   */
  validateScopes(requestedScopes) {
    const invalid = requestedScopes.filter(scope => !this.scopes.has(scope));
    
    return {
      valid: invalid.length === 0,
      invalidScopes: invalid
    };
  }
  
  /**
   * Check if token has required scope
   */
  hasScope(tokenScopes, requiredScope) {
    return tokenScopes.includes(requiredScope);
  }
  
  /**
   * Check if token has any of required scopes
   */
  hasAnyScope(tokenScopes, requiredScopes) {
    return requiredScopes.some(scope => tokenScopes.includes(scope));
  }
}

/**
 * API Key Middleware
 */
class APIKeyMiddleware {
  constructor(apiKeyService, scopeService) {
    this.apiKeyService = apiKeyService;
    this.scopeService = scopeService;
  }
  
  /**
   * Authenticate with API key
   */
  authenticate() {
    return async (req, res, next) => {
      const apiKey = this.extractAPIKey(req);
      
      if (!apiKey) {
        return res.status(401).json({
          error: 'API key required',
          message: 'Provide API key in Authorization header'
        });
      }
      
      const validation = await this.apiKeyService.validateAPIKey(apiKey);
      
      if (!validation.valid) {
        return res.status(401).json({
          error: 'Invalid API key',
          message: validation.error
        });
      }
      
      // Attach user and scopes to request
      req.apiUser = {
        userId: validation.userId,
        scopes: validation.scopes
      };
      
      next();
    };
  }
  
  /**
   * Require specific scope
   */
  requireScope(scope) {
    return (req, res, next) => {
      if (!req.apiUser) {
        return res.status(401).json({
          error: 'Authentication required'
        });
      }
      
      if (!this.scopeService.hasScope(req.apiUser.scopes, scope)) {
        return res.status(403).json({
          error: 'Insufficient scope',
          required: scope,
          provided: req.apiUser.scopes
        });
      }
      
      next();
    };
  }
  
  /**
   * Extract API key from request
   */
  extractAPIKey(req) {
    // From Authorization header: Bearer <api_key>
    const authHeader = req.headers.authorization;
    if (authHeader && authHeader.startsWith('Bearer ')) {
      return authHeader.substring(7);
    }
    
    // From X-API-Key header
    if (req.headers['x-api-key']) {
      return req.headers['x-api-key'];
    }
    
    // From query parameter (not recommended)
    if (req.query.api_key) {
      return req.query.api_key;
    }
    
    return null;
  }
}

/**
 * Rate Limiting per API Key
 */
class APIKeyRateLimiter {
  constructor() {
    this.redis = new Redis();
  }
  
  /**
   * Rate limit by API key
   */
  async checkRateLimit(apiKey, limit = 1000, windowSeconds = 3600) {
    const key = `ratelimit:apikey:${apiKey}`;
    
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, windowSeconds);
    }
    
    if (current > limit) {
      const ttl = await this.redis.ttl(key);
      
      return {
        allowed: false,
        limit,
        remaining: 0,
        resetIn: ttl
      };
    }
    
    return {
      allowed: true,
      limit,
      remaining: limit - current
    };
  }
  
  /**
   * Rate limit middleware
   */
  middleware() {
    return async (req, res, next) => {
      const apiKey = req.apiUser?.userId || req.ip;
      
      const result = await this.checkRateLimit(apiKey);
      
      // Set rate limit headers
      res.setHeader('X-RateLimit-Limit', result.limit);
      res.setHeader('X-RateLimit-Remaining', result.remaining);
      
      if (!result.allowed) {
        res.setHeader('X-RateLimit-Reset', result.resetIn);
        
        return res.status(429).json({
          error: 'Rate limit exceeded',
          limit: result.limit,
          resetIn: result.resetIn
        });
      }
      
      next();
    };
  }
}

/**
 * ENBD API with API Key Authentication
 */
class ENBDAPIWithKeys {
  constructor() {
    this.app = express();
    this.apiKeyService = new APIKeyManagementService();
    this.scopeService = new OAuthScopeService();
    this.apiKeyMiddleware = new APIKeyMiddleware(this.apiKeyService, this.scopeService);
    this.rateLimiter = new APIKeyRateLimiter();
    
    this.setupRoutes();
  }
  
  setupRoutes() {
    // Generate API key
    this.app.post('/api/keys',
      this.authenticate.bind(this),
      async (req, res) => {
        const { name, scopes } = req.body;
        
        const key = await this.apiKeyService.generateAPIKey(
          req.user.id,
          name,
          scopes
        );
        
        res.json({
          message: 'API key generated. Save it securely - it won\'t be shown again!',
          ...key
        });
      }
    );
    
    // List API keys
    this.app.get('/api/keys',
      this.authenticate.bind(this),
      async (req, res) => {
        const keys = await this.apiKeyService.listUserAPIKeys(req.user.id);
        res.json({ keys });
      }
    );
    
    // Revoke API key
    this.app.delete('/api/keys/:keyId',
      this.authenticate.bind(this),
      async (req, res) => {
        await this.apiKeyService.revokeAPIKey(req.params.keyId);
        res.json({ success: true });
      }
    );
    
    // API endpoint with API key auth
    this.app.get('/api/v1/accounts',
      this.apiKeyMiddleware.authenticate(),
      this.rateLimiter.middleware(),
      this.apiKeyMiddleware.requireScope('accounts:read'),
      async (req, res) => {
        const accounts = await this.getAccounts(req.apiUser.userId);
        res.json({ accounts });
      }
    );
  }
  
  authenticate(req, res, next) {
    req.user = { id: 'user-123' };
    next();
  }
  
  async getAccounts(userId) {
    return [];
  }
}

module.exports = {
  APIKeyManagementService,
  OAuthScopeService,
  APIKeyMiddleware,
  APIKeyRateLimiter,
  ENBDAPIWithKeys
};
```

**API Security Best Practices:**

1. ✅ **API keys for machine-to-machine**
2. ✅ **OAuth scopes for fine-grained access**
3. ✅ **Rate limiting per API key**
4. ✅ **Key rotation**
5. ✅ **Usage tracking**
6. ✅ **Expiration dates**

---

### Q30. How do you implement security monitoring and alerting?

**Answer:**

```javascript
const winston = require('winston');
const nodemailer = require('nodemailer');

/**
 * Security Monitoring Service
 */
class SecurityMonitoringService {
  constructor() {
    this.setupLogger();
    this.setupAlerting();
    this.thresholds = {
      failedLogins: 5,
      suspiciousActivity: 3,
      dataAccess: 100
    };
  }
  
  setupLogger() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.json(),
      transports: [
        new winston.transports.File({ filename: 'security.log' })
      ]
    });
  }
  
  setupAlerting() {
    this.mailer = nodemailer.createTransport({
      host: 'smtp.enbd.com',
      port: 587,
      secure: false,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS
      }
    });
  }
  
  /**
   * Monitor failed login attempts
   */
  async monitorFailedLogins(userId, ip) {
    const key = `failed_logins:${userId}:${ip}`;
    const count = await this.incrementCounter(key, 300); // 5 min window
    
    if (count >= this.thresholds.failedLogins) {
      await this.alert('CRITICAL', 'Multiple failed login attempts', {
        userId,
        ip,
        count
      });
      
      // Block IP temporarily
      await this.blockIP(ip, 3600);
    }
  }
  
  /**
   * Detect anomalies
   */
  async detectAnomaly(event) {
    const anomalies = [];
    
    // Unusual access time
    const hour = new Date().getHours();
    if (hour < 6 || hour > 22) {
      anomalies.push('Unusual access time');
    }
    
    // Rapid requests
    if (await this.isRapidRequests(event.userId)) {
      anomalies.push('Rapid requests detected');
    }
    
    // Geographic anomaly
    if (await this.isGeographicAnomaly(event.userId, event.location)) {
      anomalies.push('Unusual location');
    }
    
    if (anomalies.length > 0) {
      await this.alert('WARNING', 'Anomalous behavior detected', {
        userId: event.userId,
        anomalies
      });
    }
  }
  
  /**
   * Send security alert
   */
  async alert(severity, message, details) {
    this.logger.warn('SECURITY_ALERT', {
      severity,
      message,
      details,
      timestamp: new Date()
    });
    
    if (severity === 'CRITICAL') {
      await this.sendEmailAlert(message, details);
      await this.sendSlackAlert(message, details);
    }
  }
  
  async sendEmailAlert(message, details) {
    await this.mailer.sendMail({
      from: 'security@enbd.com',
      to: 'security-team@enbd.com',
      subject: `🚨 Security Alert: ${message}`,
      html: `
        <h2>Security Alert</h2>
        <p><strong>${message}</strong></p>
        <pre>${JSON.stringify(details, null, 2)}</pre>
      `
    });
  }
  
  async sendSlackAlert(message, details) {
    // Slack webhook integration
  }
  
  async incrementCounter(key, ttl) {
    // Redis counter
    return 1;
  }
  
  async blockIP(ip, seconds) {
    // Block IP logic
  }
  
  async isRapidRequests(userId) {
    return false;
  }
  
  async isGeographicAnomaly(userId, location) {
    return false;
  }
}

module.exports = {
  SecurityMonitoringService
};
```

**Security Monitoring Best Practices:**

1. ✅ **Real-time alerts**
2. ✅ **Failed login monitoring**
3. ✅ **Anomaly detection**
4. ✅ **Automated responses**
5. ✅ **Log aggregation**

---

**Summary Q26-Q30:**
- RBAC (roles, permissions, hierarchies) ✅
- ABAC (context-aware, fine-grained) ✅
- Secure session management (Redis, hijacking detection) ✅
- API security (API keys, OAuth scopes, rate limiting) ✅
- Security monitoring & alerting ✅

Continuing with remaining security questions...
