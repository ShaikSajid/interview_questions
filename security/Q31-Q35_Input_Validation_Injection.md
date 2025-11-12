# Security Questions 31-35: Input Validation & SQL Injection Prevention

---

### Q31. How do you prevent SQL injection attacks?

**Answer:**

```javascript
const { Pool } = require('pg');
const mysql = require('mysql2/promise');

/**
 * SQL Injection Prevention Service
 */
class SQLInjectionPreventionService {
  constructor() {
    // PostgreSQL connection pool
    this.pgPool = new Pool({
      host: 'localhost',
      database: 'enbd',
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      max: 20
    });
    
    // MySQL connection pool
    this.mysqlPool = mysql.createPool({
      host: 'localhost',
      database: 'enbd',
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      waitForConnections: true,
      connectionLimit: 10
    });
  }
  
  /**
   * ✅ SAFE: Using parameterized queries (PostgreSQL)
   */
  async safeQueryPG(userId) {
    const query = 'SELECT * FROM users WHERE id = $1';
    const values = [userId];
    
    const result = await this.pgPool.query(query, values);
    return result.rows;
  }
  
  /**
   * ✅ SAFE: Using parameterized queries (MySQL)
   */
  async safeQueryMySQL(email) {
    const query = 'SELECT * FROM users WHERE email = ?';
    const [rows] = await this.mysqlPool.execute(query, [email]);
    return rows;
  }
  
  /**
   * ❌ VULNERABLE: String concatenation
   */
  async vulnerableQuery(username) {
    // NEVER DO THIS!
    const query = `SELECT * FROM users WHERE username = '${username}'`;
    // Attacker input: ' OR '1'='1' --
    // Results in: SELECT * FROM users WHERE username = '' OR '1'='1' --'
    
    // This would return ALL users!
    const result = await this.pgPool.query(query);
    return result.rows;
  }
  
  /**
   * ✅ SAFE: Complex query with multiple parameters
   */
  async safeComplexQuery(filters) {
    const { accountType, minBalance, status } = filters;
    
    const query = `
      SELECT * FROM accounts 
      WHERE account_type = $1 
        AND balance >= $2 
        AND status = $3
      ORDER BY created_at DESC
      LIMIT 100
    `;
    
    const values = [accountType, minBalance, status];
    
    const result = await this.pgPool.query(query, values);
    return result.rows;
  }
  
  /**
   * ✅ SAFE: Dynamic WHERE clause with parameterized queries
   */
  async safeDynamicQuery(filters) {
    let query = 'SELECT * FROM transactions WHERE 1=1';
    const values = [];
    let paramIndex = 1;
    
    if (filters.accountId) {
      query += ` AND account_id = $${paramIndex}`;
      values.push(filters.accountId);
      paramIndex++;
    }
    
    if (filters.startDate) {
      query += ` AND created_at >= $${paramIndex}`;
      values.push(filters.startDate);
      paramIndex++;
    }
    
    if (filters.endDate) {
      query += ` AND created_at <= $${paramIndex}`;
      values.push(filters.endDate);
      paramIndex++;
    }
    
    if (filters.minAmount) {
      query += ` AND amount >= $${paramIndex}`;
      values.push(filters.minAmount);
      paramIndex++;
    }
    
    query += ' ORDER BY created_at DESC LIMIT 100';
    
    const result = await this.pgPool.query(query, values);
    return result.rows;
  }
  
  /**
   * ✅ SAFE: Using ORM (Sequelize example)
   */
  async safeORMQuery(Sequelize, User) {
    // Sequelize automatically uses parameterized queries
    const users = await User.findAll({
      where: {
        email: 'user@enbd.com',
        status: 'active'
      },
      attributes: ['id', 'email', 'name'],
      limit: 10
    });
    
    return users;
  }
  
  /**
   * ✅ SAFE: Stored procedures
   */
  async safeStoredProcedure(userId, amount) {
    const query = 'CALL transfer_funds($1, $2)';
    const values = [userId, amount];
    
    const result = await this.pgPool.query(query, values);
    return result.rows;
  }
  
  /**
   * ✅ SAFE: Input sanitization (as additional layer)
   */
  sanitizeInput(input) {
    if (typeof input !== 'string') {
      return input;
    }
    
    // Remove dangerous characters
    return input
      .replace(/[^\w\s@.-]/g, '') // Allow only word chars, spaces, @, ., -
      .trim()
      .substring(0, 255); // Limit length
  }
  
  /**
   * ✅ SAFE: Whitelist approach for column names
   */
  async safeSortQuery(sortColumn, sortOrder) {
    // Whitelist allowed columns
    const allowedColumns = ['name', 'email', 'created_at', 'balance'];
    const allowedOrders = ['ASC', 'DESC'];
    
    if (!allowedColumns.includes(sortColumn)) {
      throw new Error('Invalid sort column');
    }
    
    if (!allowedOrders.includes(sortOrder.toUpperCase())) {
      throw new Error('Invalid sort order');
    }
    
    // Safe to use string interpolation for whitelisted values
    const query = `SELECT * FROM users ORDER BY ${sortColumn} ${sortOrder}`;
    
    const result = await this.pgPool.query(query);
    return result.rows;
  }
  
  /**
   * ✅ SAFE: Query builder pattern
   */
  buildSafeQuery(tableName, conditions) {
    // Whitelist table names
    const allowedTables = ['users', 'accounts', 'transactions'];
    
    if (!allowedTables.includes(tableName)) {
      throw new Error('Invalid table name');
    }
    
    let query = `SELECT * FROM ${tableName} WHERE 1=1`;
    const values = [];
    let paramIndex = 1;
    
    for (const [key, value] of Object.entries(conditions)) {
      // Whitelist column names
      const allowedColumns = this.getAllowedColumns(tableName);
      
      if (!allowedColumns.includes(key)) {
        throw new Error(`Invalid column: ${key}`);
      }
      
      query += ` AND ${key} = $${paramIndex}`;
      values.push(value);
      paramIndex++;
    }
    
    return { query, values };
  }
  
  getAllowedColumns(tableName) {
    const columnWhitelist = {
      users: ['id', 'email', 'name', 'status'],
      accounts: ['id', 'user_id', 'account_number', 'balance'],
      transactions: ['id', 'account_id', 'amount', 'type', 'status']
    };
    
    return columnWhitelist[tableName] || [];
  }
}

/**
 * MongoDB NoSQL Injection Prevention
 */
class NoSQLInjectionPreventionService {
  constructor() {
    this.mongoose = require('mongoose');
  }
  
  /**
   * ❌ VULNERABLE: Direct user input
   */
  async vulnerableMongoQuery(username) {
    // NEVER DO THIS!
    // Attacker input: { $gt: "" } as JSON
    // Results in: db.users.find({ username: { $gt: "" } })
    // Returns ALL users!
    
    const User = this.mongoose.model('User');
    const users = await User.find({ username });
    return users;
  }
  
  /**
   * ✅ SAFE: Type validation
   */
  async safeMongoQuery(username) {
    // Ensure username is a string
    if (typeof username !== 'string') {
      throw new Error('Invalid username type');
    }
    
    const User = this.mongoose.model('User');
    const users = await User.find({ username });
    return users;
  }
  
  /**
   * ✅ SAFE: Input sanitization for MongoDB
   */
  sanitizeMongoInput(input) {
    if (typeof input === 'object' && input !== null) {
      // Remove $ operators
      const sanitized = {};
      for (const [key, value] of Object.entries(input)) {
        if (!key.startsWith('$')) {
          sanitized[key] = this.sanitizeMongoInput(value);
        }
      }
      return sanitized;
    }
    
    return input;
  }
  
  /**
   * ✅ SAFE: Using mongoose schema validation
   */
  createUserSchema() {
    const userSchema = new this.mongoose.Schema({
      email: {
        type: String,
        required: true,
        validate: {
          validator: (v) => /^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/.test(v),
          message: 'Invalid email format'
        }
      },
      age: {
        type: Number,
        min: 18,
        max: 120
      },
      role: {
        type: String,
        enum: ['customer', 'teller', 'manager', 'admin']
      }
    });
    
    return this.mongoose.model('User', userSchema);
  }
}

/**
 * ENBD Banking Query Service (Secure)
 */
class ENBDBankingQueryService {
  constructor() {
    this.sqlService = new SQLInjectionPreventionService();
  }
  
  /**
   * Get user accounts (secure)
   */
  async getUserAccounts(userId) {
    const query = `
      SELECT 
        id,
        account_number,
        account_type,
        balance,
        currency,
        status,
        created_at
      FROM accounts
      WHERE user_id = $1 AND status = 'active'
      ORDER BY created_at DESC
    `;
    
    return await this.sqlService.pgPool.query(query, [userId]);
  }
  
  /**
   * Search transactions (secure with dynamic filters)
   */
  async searchTransactions(filters) {
    // Validate inputs
    this.validateTransactionFilters(filters);
    
    let query = `
      SELECT 
        t.id,
        t.amount,
        t.currency,
        t.type,
        t.status,
        t.created_at,
        a.account_number
      FROM transactions t
      JOIN accounts a ON t.account_id = a.id
      WHERE 1=1
    `;
    
    const values = [];
    let paramIndex = 1;
    
    if (filters.accountId) {
      query += ` AND t.account_id = $${paramIndex}`;
      values.push(filters.accountId);
      paramIndex++;
    }
    
    if (filters.startDate) {
      query += ` AND t.created_at >= $${paramIndex}`;
      values.push(filters.startDate);
      paramIndex++;
    }
    
    if (filters.endDate) {
      query += ` AND t.created_at <= $${paramIndex}`;
      values.push(filters.endDate);
      paramIndex++;
    }
    
    if (filters.minAmount) {
      query += ` AND t.amount >= $${paramIndex}`;
      values.push(filters.minAmount);
      paramIndex++;
    }
    
    if (filters.maxAmount) {
      query += ` AND t.amount <= $${paramIndex}`;
      values.push(filters.maxAmount);
      paramIndex++;
    }
    
    if (filters.type) {
      query += ` AND t.type = $${paramIndex}`;
      values.push(filters.type);
      paramIndex++;
    }
    
    query += ` ORDER BY t.created_at DESC LIMIT 100`;
    
    return await this.sqlService.pgPool.query(query, values);
  }
  
  /**
   * Validate transaction filters
   */
  validateTransactionFilters(filters) {
    if (filters.accountId && typeof filters.accountId !== 'number') {
      throw new Error('Invalid accountId');
    }
    
    if (filters.startDate && !(filters.startDate instanceof Date)) {
      throw new Error('Invalid startDate');
    }
    
    if (filters.type) {
      const allowedTypes = ['transfer', 'payment', 'withdrawal', 'deposit'];
      if (!allowedTypes.includes(filters.type)) {
        throw new Error('Invalid transaction type');
      }
    }
    
    if (filters.minAmount && (typeof filters.minAmount !== 'number' || filters.minAmount < 0)) {
      throw new Error('Invalid minAmount');
    }
  }
  
  /**
   * Transfer funds (secure with stored procedure)
   */
  async transferFunds(fromAccount, toAccount, amount) {
    // Input validation
    if (!Number.isInteger(fromAccount) || !Number.isInteger(toAccount)) {
      throw new Error('Invalid account IDs');
    }
    
    if (typeof amount !== 'number' || amount <= 0 || amount > 1000000) {
      throw new Error('Invalid amount');
    }
    
    // Use stored procedure for complex operations
    const query = 'CALL sp_transfer_funds($1, $2, $3)';
    const values = [fromAccount, toAccount, amount];
    
    return await this.sqlService.pgPool.query(query, values);
  }
}

/**
 * Database Security Best Practices
 */
class DatabaseSecurityBestPractices {
  /**
   * Connection security
   */
  static getSecureConnection() {
    return {
      // ✅ Use connection pooling
      max: 20,
      min: 5,
      
      // ✅ Use SSL/TLS
      ssl: {
        rejectUnauthorized: true,
        ca: fs.readFileSync('./certs/ca.pem'),
        cert: fs.readFileSync('./certs/client-cert.pem'),
        key: fs.readFileSync('./certs/client-key.pem')
      },
      
      // ✅ Use least privilege user
      user: 'enbd_app_user', // Not 'root' or 'admin'
      
      // ✅ Use environment variables for credentials
      password: process.env.DB_PASSWORD,
      
      // ✅ Set connection timeout
      connectionTimeoutMillis: 2000,
      
      // ✅ Set query timeout
      query_timeout: 10000
    };
  }
  
  /**
   * Principle of least privilege
   */
  static async setupDatabaseUser(client) {
    // Create app user with minimal permissions
    await client.query(`
      CREATE USER enbd_app_user WITH PASSWORD 'secure_password';
      
      -- Grant only necessary permissions
      GRANT SELECT, INSERT, UPDATE ON accounts TO enbd_app_user;
      GRANT SELECT, INSERT ON transactions TO enbd_app_user;
      GRANT SELECT ON users TO enbd_app_user;
      
      -- Deny dangerous permissions
      REVOKE DELETE ON ALL TABLES FROM enbd_app_user;
      REVOKE DROP ON ALL TABLES FROM enbd_app_user;
      REVOKE ALTER ON ALL TABLES FROM enbd_app_user;
    `);
  }
  
  /**
   * Enable query logging for security audit
   */
  static configureQueryLogging() {
    return {
      // Log all queries in development
      logging: process.env.NODE_ENV === 'development' ? console.log : false,
      
      // Log slow queries in production
      benchmark: true,
      logQueryParameters: false, // Don't log sensitive parameters
      
      // Custom logger that filters sensitive data
      customLogger: (query, time) => {
        if (time > 1000) {
          console.warn(`Slow query (${time}ms):`, query);
        }
      }
    };
  }
}

module.exports = {
  SQLInjectionPreventionService,
  NoSQLInjectionPreventionService,
  ENBDBankingQueryService,
  DatabaseSecurityBestPractices
};
```

**SQL Injection Prevention Checklist:**

1. ✅ **Always use parameterized queries/prepared statements**
2. ✅ **Never concatenate user input into SQL**
3. ✅ **Whitelist column names and table names**
4. ✅ **Validate and sanitize all inputs**
5. ✅ **Use ORM with proper escaping**
6. ✅ **Principle of least privilege for DB users**
7. ✅ **Use stored procedures for complex operations**
8. ✅ **Enable SSL/TLS for database connections**

---

### Q32. How do you implement comprehensive input validation?

**Answer:**

```javascript
const validator = require('validator');
const Joi = require('joi');

/**
 * Input Validation Service
 */
class InputValidationService {
  /**
   * Validate email
   */
  validateEmail(email) {
    if (!validator.isEmail(email)) {
      return {
        valid: false,
        error: 'Invalid email format'
      };
    }
    
    // Additional checks
    if (email.length > 255) {
      return {
        valid: false,
        error: 'Email too long'
      };
    }
    
    // Check disposable email domains
    const disposableDomains = ['tempmail.com', '10minutemail.com'];
    const domain = email.split('@')[1];
    
    if (disposableDomains.includes(domain)) {
      return {
        valid: false,
        error: 'Disposable email addresses not allowed'
      };
    }
    
    return { valid: true };
  }
  
  /**
   * Validate phone number (UAE format)
   */
  validatePhoneNumber(phone) {
    // UAE phone numbers: +971 XX XXX XXXX
    const uaePhoneRegex = /^\+971(50|52|54|55|56|58)\d{7}$/;
    
    if (!uaePhoneRegex.test(phone)) {
      return {
        valid: false,
        error: 'Invalid UAE phone number format'
      };
    }
    
    return { valid: true };
  }
  
  /**
   * Validate IBAN (UAE)
   */
  validateIBAN(iban) {
    // UAE IBAN: AE07 0331 2345 6789 0123 456
    if (!validator.isIBAN(iban)) {
      return {
        valid: false,
        error: 'Invalid IBAN format'
      };
    }
    
    if (!iban.startsWith('AE')) {
      return {
        valid: false,
        error: 'Only UAE IBANs accepted'
      };
    }
    
    return { valid: true };
  }
  
  /**
   * Validate amount
   */
  validateAmount(amount, options = {}) {
    const {
      min = 0.01,
      max = 1000000,
      currency = 'AED'
    } = options;
    
    if (typeof amount !== 'number' || isNaN(amount)) {
      return {
        valid: false,
        error: 'Amount must be a number'
      };
    }
    
    if (amount < min) {
      return {
        valid: false,
        error: `Amount must be at least ${min} ${currency}`
      };
    }
    
    if (amount > max) {
      return {
        valid: false,
        error: `Amount cannot exceed ${max} ${currency}`
      };
    }
    
    // Check decimal places
    if (!Number.isInteger(amount * 100)) {
      return {
        valid: false,
        error: 'Amount can have maximum 2 decimal places'
      };
    }
    
    return { valid: true };
  }
  
  /**
   * Validate date
   */
  validateDate(date, options = {}) {
    const {
      format = 'YYYY-MM-DD',
      minDate = null,
      maxDate = null
    } = options;
    
    if (!validator.isDate(date)) {
      return {
        valid: false,
        error: 'Invalid date format'
      };
    }
    
    const dateObj = new Date(date);
    
    if (minDate && dateObj < new Date(minDate)) {
      return {
        valid: false,
        error: `Date must be after ${minDate}`
      };
    }
    
    if (maxDate && dateObj > new Date(maxDate)) {
      return {
        valid: false,
        error: `Date must be before ${maxDate}`
      };
    }
    
    return { valid: true };
  }
  
  /**
   * Validate URL
   */
  validateURL(url) {
    if (!validator.isURL(url, {
      protocols: ['https'],
      require_protocol: true,
      require_valid_protocol: true
    })) {
      return {
        valid: false,
        error: 'Invalid HTTPS URL'
      };
    }
    
    // Check domain whitelist
    const allowedDomains = ['enbd.com', 'emiratesnbd.com'];
    const urlObj = new URL(url);
    
    if (!allowedDomains.some(domain => urlObj.hostname.endsWith(domain))) {
      return {
        valid: false,
        error: 'URL domain not allowed'
      };
    }
    
    return { valid: true };
  }
  
  /**
   * Sanitize string input
   */
  sanitizeString(input, options = {}) {
    const {
      maxLength = 255,
      allowedChars = /^[a-zA-Z0-9\s\-_.,@]+$/
    } = options;
    
    if (typeof input !== 'string') {
      return '';
    }
    
    // Trim whitespace
    let sanitized = input.trim();
    
    // Remove dangerous characters
    sanitized = validator.escape(sanitized);
    
    // Limit length
    sanitized = sanitized.substring(0, maxLength);
    
    // Check allowed characters
    if (!allowedChars.test(sanitized)) {
      throw new Error('Input contains invalid characters');
    }
    
    return sanitized;
  }
  
  /**
   * Validate array
   */
  validateArray(arr, options = {}) {
    const {
      minLength = 0,
      maxLength = 100,
      itemValidator = null
    } = options;
    
    if (!Array.isArray(arr)) {
      return {
        valid: false,
        error: 'Input must be an array'
      };
    }
    
    if (arr.length < minLength) {
      return {
        valid: false,
        error: `Array must have at least ${minLength} items`
      };
    }
    
    if (arr.length > maxLength) {
      return {
        valid: false,
        error: `Array cannot have more than ${maxLength} items`
      };
    }
    
    if (itemValidator) {
      for (let i = 0; i < arr.length; i++) {
        const result = itemValidator(arr[i]);
        if (!result.valid) {
          return {
            valid: false,
            error: `Item ${i}: ${result.error}`
          };
        }
      }
    }
    
    return { valid: true };
  }
}

/**
 * Joi Schema Validation
 */
class JoiValidationService {
  /**
   * User registration schema
   */
  static getUserRegistrationSchema() {
    return Joi.object({
      email: Joi.string()
        .email()
        .required()
        .messages({
          'string.email': 'Invalid email format',
          'any.required': 'Email is required'
        }),
      
      password: Joi.string()
        .min(12)
        .max(128)
        .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/)
        .required()
        .messages({
          'string.min': 'Password must be at least 12 characters',
          'string.pattern.base': 'Password must contain uppercase, lowercase, number, and special character'
        }),
      
      firstName: Joi.string()
        .min(2)
        .max(50)
        .pattern(/^[a-zA-Z\s]+$/)
        .required(),
      
      lastName: Joi.string()
        .min(2)
        .max(50)
        .pattern(/^[a-zA-Z\s]+$/)
        .required(),
      
      phoneNumber: Joi.string()
        .pattern(/^\+971(50|52|54|55|56|58)\d{7}$/)
        .required()
        .messages({
          'string.pattern.base': 'Invalid UAE phone number'
        }),
      
      dateOfBirth: Joi.date()
        .max('now')
        .min('1900-01-01')
        .required()
        .messages({
          'date.max': 'Date of birth cannot be in the future',
          'date.min': 'Invalid date of birth'
        }),
      
      emiratesId: Joi.string()
        .pattern(/^\d{3}-\d{4}-\d{7}-\d{1}$/)
        .required()
        .messages({
          'string.pattern.base': 'Invalid Emirates ID format (XXX-XXXX-XXXXXXX-X)'
        }),
      
      address: Joi.object({
        street: Joi.string().required(),
        city: Joi.string().required(),
        emirate: Joi.string()
          .valid('Abu Dhabi', 'Dubai', 'Sharjah', 'Ajman', 'Umm Al Quwain', 'Ras Al Khaimah', 'Fujairah')
          .required(),
        poBox: Joi.string().pattern(/^\d+$/).optional()
      }).required()
    });
  }
  
  /**
   * Transaction schema
   */
  static getTransactionSchema() {
    return Joi.object({
      fromAccount: Joi.string()
        .pattern(/^\d{16}$/)
        .required()
        .messages({
          'string.pattern.base': 'Invalid account number format'
        }),
      
      toAccount: Joi.string()
        .pattern(/^\d{16}$/)
        .required(),
      
      amount: Joi.number()
        .positive()
        .precision(2)
        .min(0.01)
        .max(1000000)
        .required()
        .messages({
          'number.positive': 'Amount must be positive',
          'number.min': 'Minimum transfer amount is 0.01 AED',
          'number.max': 'Maximum transfer amount is 1,000,000 AED'
        }),
      
      currency: Joi.string()
        .valid('AED', 'USD', 'EUR', 'GBP')
        .default('AED'),
      
      reference: Joi.string()
        .max(140)
        .pattern(/^[a-zA-Z0-9\s\-.,]+$/)
        .optional(),
      
      scheduledDate: Joi.date()
        .min('now')
        .max(Joi.ref('$maxScheduleDate'))
        .optional()
        .messages({
          'date.min': 'Scheduled date cannot be in the past',
          'date.max': 'Cannot schedule more than 1 year in advance'
        })
    });
  }
  
  /**
   * Validate data against schema
   */
  static validate(data, schema) {
    const result = schema.validate(data, {
      abortEarly: false, // Return all errors
      stripUnknown: true // Remove unknown fields
    });
    
    if (result.error) {
      const errors = result.error.details.map(err => ({
        field: err.path.join('.'),
        message: err.message
      }));
      
      return {
        valid: false,
        errors
      };
    }
    
    return {
      valid: true,
      data: result.value
    };
  }
}

/**
 * Validation Middleware
 */
class ValidationMiddleware {
  /**
   * Validate request body
   */
  static validateBody(schema) {
    return (req, res, next) => {
      const result = JoiValidationService.validate(req.body, schema);
      
      if (!result.valid) {
        return res.status(400).json({
          error: 'Validation failed',
          details: result.errors
        });
      }
      
      // Replace req.body with validated data
      req.body = result.data;
      next();
    };
  }
  
  /**
   * Validate query parameters
   */
  static validateQuery(schema) {
    return (req, res, next) => {
      const result = JoiValidationService.validate(req.query, schema);
      
      if (!result.valid) {
        return res.status(400).json({
          error: 'Validation failed',
          details: result.errors
        });
      }
      
      req.query = result.data;
      next();
    };
  }
  
  /**
   * Validate path parameters
   */
  static validateParams(schema) {
    return (req, res, next) => {
      const result = JoiValidationService.validate(req.params, schema);
      
      if (!result.valid) {
        return res.status(400).json({
          error: 'Validation failed',
          details: result.errors
        });
      }
      
      req.params = result.data;
      next();
    };
  }
}

/**
 * ENBD Banking Input Validation Example
 */
class ENBDBankingInputValidation {
  setupRoutes(app) {
    // User registration with validation
    app.post('/api/auth/register',
      ValidationMiddleware.validateBody(JoiValidationService.getUserRegistrationSchema()),
      async (req, res) => {
        // req.body is now validated and sanitized
        const user = await this.createUser(req.body);
        res.json({ success: true, userId: user.id });
      }
    );
    
    // Transaction with validation
    app.post('/api/transactions',
      ValidationMiddleware.validateBody(JoiValidationService.getTransactionSchema()),
      async (req, res) => {
        const transaction = await this.createTransaction(req.body);
        res.json({ success: true, transactionId: transaction.id });
      }
    );
    
    // Search with query validation
    const searchSchema = Joi.object({
      q: Joi.string().min(3).max(100).required(),
      page: Joi.number().integer().min(1).default(1),
      limit: Joi.number().integer().min(1).max(100).default(20)
    });
    
    app.get('/api/search',
      ValidationMiddleware.validateQuery(searchSchema),
      async (req, res) => {
        const results = await this.search(req.query);
        res.json({ results });
      }
    );
  }
  
  async createUser(data) {
    return { id: 'user-123' };
  }
  
  async createTransaction(data) {
    return { id: 'txn-123' };
  }
  
  async search(params) {
    return [];
  }
}

module.exports = {
  InputValidationService,
  JoiValidationService,
  ValidationMiddleware,
  ENBDBankingInputValidation
};
```

**Input Validation Best Practices:**

1. ✅ **Validate all inputs** (body, query, params, headers)
2. ✅ **Use schema validation** (Joi, Yup, etc.)
3. ✅ **Whitelist approach** (define what's allowed)
4. ✅ **Type checking** (ensure correct data types)
5. ✅ **Length limits** (prevent buffer overflow)
6. ✅ **Format validation** (regex for patterns)
7. ✅ **Sanitization** (remove dangerous characters)
8. ✅ **Business logic validation** (amount limits, date ranges)

---

### Q33. How do you prevent command injection and file upload vulnerabilities?

**Answer:**

```javascript
const multer = require('multer');
const crypto = require('crypto');
const path = require('path');
const { exec } = require('child_process');
const { promisify } = require('util');
const execAsync = promisify(exec);

/**
 * Command Injection Prevention Service
 */
class CommandInjectionPreventionService {
  /**
   * ❌ VULNERABLE: Direct command execution
   */
  async vulnerableCommand(filename) {
    // NEVER DO THIS!
    // Attacker input: "file.txt; rm -rf /"
    const command = `cat ${filename}`;
    await execAsync(command);
  }
  
  /**
   * ✅ SAFE: Use array-based arguments
   */
  async safeCommand(filename) {
    const { spawn } = require('child_process');
    
    return new Promise((resolve, reject) => {
      const process = spawn('cat', [filename]); // Arguments as array
      
      let output = '';
      
      process.stdout.on('data', (data) => {
        output += data.toString();
      });
      
      process.on('close', (code) => {
        if (code === 0) {
          resolve(output);
        } else {
          reject(new Error(`Command failed with code ${code}`));
        }
      });
      
      process.on('error', reject);
    });
  }
  
  /**
   * ✅ SAFE: Input validation and whitelisting
   */
  async safeCommandWithValidation(filename) {
    // Validate filename
    if (!this.isValidFilename(filename)) {
      throw new Error('Invalid filename');
    }
    
    // Whitelist allowed commands
    const allowedCommands = ['cat', 'ls', 'head'];
    const command = 'cat';
    
    if (!allowedCommands.includes(command)) {
      throw new Error('Command not allowed');
    }
    
    const { spawn } = require('child_process');
    const process = spawn(command, [filename]);
    
    return new Promise((resolve, reject) => {
      let output = '';
      process.stdout.on('data', (data) => output += data);
      process.on('close', (code) => {
        code === 0 ? resolve(output) : reject(new Error('Command failed'));
      });
    });
  }
  
  /**
   * Validate filename
   */
  isValidFilename(filename) {
    // No path traversal
    if (filename.includes('..') || filename.includes('/') || filename.includes('\\')) {
      return false;
    }
    
    // Only alphanumeric, dash, underscore, dot
    if (!/^[a-zA-Z0-9\-_.]+$/.test(filename)) {
      return false;
    }
    
    return true;
  }
  
  /**
   * ✅ SAFE: Escape shell arguments
   */
  escapeShellArg(arg) {
    return `'${arg.replace(/'/g, "'\\''")}'`;
  }
}

/**
 * Secure File Upload Service
 */
class SecureFileUploadService {
  constructor() {
    this.uploadDir = './uploads';
    this.maxFileSize = 5 * 1024 * 1024; // 5MB
    this.allowedMimeTypes = [
      'image/jpeg',
      'image/png',
      'image/gif',
      'application/pdf',
      'text/plain'
    ];
    this.allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif', '.pdf', '.txt'];
  }
  
  /**
   * Configure multer with security settings
   */
  getUploadMiddleware() {
    const storage = multer.diskStorage({
      destination: (req, file, cb) => {
        // Store in isolated directory
        cb(null, this.uploadDir);
      },
      
      filename: (req, file, cb) => {
        // Generate random filename (don't use original name)
        const randomName = crypto.randomBytes(16).toString('hex');
        const ext = path.extname(file.originalname).toLowerCase();
        cb(null, `${randomName}${ext}`);
      }
    });
    
    return multer({
      storage,
      
      limits: {
        fileSize: this.maxFileSize,
        files: 1 // Max number of files
      },
      
      fileFilter: (req, file, cb) => {
        // Validate file type
        if (!this.isAllowedFile(file)) {
          return cb(new Error('File type not allowed'));
        }
        
        cb(null, true);
      }
    });
  }
  
  /**
   * Validate file type
   */
  isAllowedFile(file) {
    // Check MIME type
    if (!this.allowedMimeTypes.includes(file.mimetype)) {
      return false;
    }
    
    // Check extension
    const ext = path.extname(file.originalname).toLowerCase();
    if (!this.allowedExtensions.includes(ext)) {
      return false;
    }
    
    return true;
  }
  
  /**
   * Validate file content (magic bytes)
   */
  async validateFileContent(filepath) {
    const fs = require('fs').promises;
    const buffer = await fs.readFile(filepath);
    
    // Check magic bytes for common file types
    const magicBytes = {
      'image/jpeg': [0xFF, 0xD8, 0xFF],
      'image/png': [0x89, 0x50, 0x4E, 0x47],
      'image/gif': [0x47, 0x49, 0x46],
      'application/pdf': [0x25, 0x50, 0x44, 0x46] // %PDF
    };
    
    for (const [mimeType, bytes] of Object.entries(magicBytes)) {
      if (this.matchesMagicBytes(buffer, bytes)) {
        return { valid: true, mimeType };
      }
    }
    
    return {
      valid: false,
      error: 'File content does not match declared type'
    };
  }
  
  matchesMagicBytes(buffer, magicBytes) {
    for (let i = 0; i < magicBytes.length; i++) {
      if (buffer[i] !== magicBytes[i]) {
        return false;
      }
    }
    return true;
  }
  
  /**
   * Scan file for malware (using ClamAV)
   */
  async scanForMalware(filepath) {
    try {
      const { spawn } = require('child_process');
      const clamscan = spawn('clamscan', ['--no-summary', filepath]);
      
      return new Promise((resolve, reject) => {
        let output = '';
        
        clamscan.stdout.on('data', (data) => {
          output += data.toString();
        });
        
        clamscan.on('close', (code) => {
          // code 0 = clean, code 1 = infected
          if (code === 0) {
            resolve({ clean: true });
          } else if (code === 1) {
            resolve({ clean: false, reason: 'Malware detected' });
          } else {
            reject(new Error('Scan failed'));
          }
        });
      });
    } catch (error) {
      console.error('Malware scan failed:', error);
      return { clean: false, reason: 'Scan unavailable' };
    }
  }
  
  /**
   * Sanitize filename
   */
  sanitizeFilename(filename) {
    return filename
      .replace(/[^a-zA-Z0-9.-]/g, '_') // Replace unsafe chars
      .replace(/\.+/g, '.') // Remove multiple dots
      .substring(0, 255); // Limit length
  }
  
  /**
   * Store file metadata securely
   */
  async storeFileMetadata(file, userId) {
    return {
      id: crypto.randomBytes(16).toString('hex'),
      userId,
      originalName: this.sanitizeFilename(file.originalname),
      storedName: file.filename,
      mimeType: file.mimetype,
      size: file.size,
      uploadedAt: new Date(),
      scanStatus: 'pending'
    };
  }
}

/**
 * ENBD Document Upload Service
 */
class ENBDDocumentUploadService {
  constructor() {
    this.uploadService = new SecureFileUploadService();
  }
  
  setupRoutes(app) {
    // Upload document
    app.post('/api/documents/upload',
      this.authenticate,
      this.uploadService.getUploadMiddleware().single('document'),
      async (req, res) => {
        try {
          // Validate file content
          const contentValidation = await this.uploadService.validateFileContent(req.file.path);
          
          if (!contentValidation.valid) {
            await this.deleteFile(req.file.path);
            return res.status(400).json({
              error: contentValidation.error
            });
          }
          
          // Scan for malware
          const scanResult = await this.uploadService.scanForMalware(req.file.path);
          
          if (!scanResult.clean) {
            await this.deleteFile(req.file.path);
            return res.status(400).json({
              error: 'File failed security scan',
              reason: scanResult.reason
            });
          }
          
          // Store metadata
          const metadata = await this.uploadService.storeFileMetadata(req.file, req.user.id);
          
          res.json({
            success: true,
            documentId: metadata.id,
            filename: metadata.originalName
          });
          
        } catch (error) {
          console.error('Upload error:', error);
          res.status(500).json({
            error: 'Upload failed'
          });
        }
      }
    );
    
    // Download document (with access control)
    app.get('/api/documents/:id/download',
      this.authenticate,
      async (req, res) => {
        const document = await this.getDocument(req.params.id);
        
        // Check ownership
        if (document.userId !== req.user.id) {
          return res.status(403).json({
            error: 'Access denied'
          });
        }
        
        // Set security headers
        res.setHeader('Content-Type', document.mimeType);
        res.setHeader('Content-Disposition', `attachment; filename="${document.originalName}"`);
        res.setHeader('X-Content-Type-Options', 'nosniff');
        
        // Stream file
        const filepath = path.join(this.uploadService.uploadDir, document.storedName);
        res.sendFile(filepath);
      }
    );
  }
  
  authenticate(req, res, next) {
    req.user = { id: 'user-123' };
    next();
  }
  
  async getDocument(id) {
    return {
      id,
      userId: 'user-123',
      originalName: 'document.pdf',
      storedName: 'abc123.pdf',
      mimeType: 'application/pdf'
    };
  }
  
  async deleteFile(filepath) {
    const fs = require('fs').promises;
    await fs.unlink(filepath);
  }
}

/**
 * Path Traversal Prevention
 */
class PathTraversalPreventionService {
  /**
   * Validate file path
   */
  isValidPath(userPath, baseDir) {
    // Resolve absolute path
    const resolvedPath = path.resolve(baseDir, userPath);
    
    // Ensure it's within baseDir
    if (!resolvedPath.startsWith(baseDir)) {
      return false;
    }
    
    return true;
  }
  
  /**
   * Sanitize path
   */
  sanitizePath(userPath) {
    return userPath
      .replace(/\.\./g, '') // Remove ..
      .replace(/[\/\\]/g, '_'); // Remove slashes
  }
  
  /**
   * Safe file access
   */
  async safeFileAccess(filename, baseDir) {
    const safePath = path.join(baseDir, path.basename(filename));
    
    if (!this.isValidPath(safePath, baseDir)) {
      throw new Error('Invalid file path');
    }
    
    const fs = require('fs').promises;
    return await fs.readFile(safePath, 'utf8');
  }
}

module.exports = {
  CommandInjectionPreventionService,
  SecureFileUploadService,
  ENBDDocumentUploadService,
  PathTraversalPreventionService
};
```

**File Upload Security Checklist:**

1. ✅ **Validate file type** (MIME type + extension)
2. ✅ **Check magic bytes** (verify actual content)
3. ✅ **Limit file size** (prevent DoS)
4. ✅ **Random filenames** (don't use original)
5. ✅ **Scan for malware** (ClamAV, VirusTotal)
6. ✅ **Isolated storage** (separate directory)
7. ✅ **Access control** (verify ownership)
8. ✅ **No execution permissions** (chmod 644)

---

### Q34. How do you prevent XXE (XML External Entity) attacks?

**Answer:**

```javascript
const xml2js = require('xml2js');
const { DOMParser } = require('xmldom');
const libxmljs = require('libxmljs');

/**
 * XXE Prevention Service
 */
class XXEPreventionService {
  /**
   * ❌ VULNERABLE: Default XML parsing
   */
  async vulnerableXMLParse(xmlString) {
    // NEVER DO THIS!
    const parser = new DOMParser();
    const doc = parser.parseFromString(xmlString, 'text/xml');
    // External entities will be processed!
  }
  
  /**
   * ✅ SAFE: xml2js with secure options
   */
  async safeXML2JSParse(xmlString) {
    const parser = new xml2js.Parser({
      // Disable external entities
      xmlns: false,
      explicitArray: false,
      // Don't allow DOCTYPE declarations
      strict: true
    });
    
    try {
      const result = await parser.parseStringPromise(xmlString);
      return result;
    } catch (error) {
      throw new Error('Invalid XML');
    }
  }
  
  /**
   * ✅ SAFE: libxmljs with secure options
   */
  safeLibxmljsParse(xmlString) {
    try {
      const doc = libxmljs.parseXml(xmlString, {
        noent: false,  // Disable entity substitution
        nonet: true,   // Disable network access
        dtdload: false, // Disable DTD loading
        dtdvalid: false // Disable DTD validation
      });
      
      return doc;
    } catch (error) {
      throw new Error('Invalid XML');
    }
  }
  
  /**
   * Validate XML before parsing
   */
  validateXML(xmlString) {
    // Check for dangerous patterns
    const dangerousPatterns = [
      /<!DOCTYPE/i,
      /<!ENTITY/i,
      /SYSTEM/i,
      /PUBLIC/i,
      /file:\/\//i,
      /http:\/\//i,
      /https:\/\//i
    ];
    
    for (const pattern of dangerousPatterns) {
      if (pattern.test(xmlString)) {
        return {
          valid: false,
          error: 'XML contains potentially dangerous content'
        };
      }
    }
    
    return { valid: true };
  }
  
  /**
   * Sanitize XML
   */
  sanitizeXML(xmlString) {
    // Remove DOCTYPE declarations
    xmlString = xmlString.replace(/<!DOCTYPE[^>]*>/gi, '');
    
    // Remove ENTITY declarations
    xmlString = xmlString.replace(/<!ENTITY[^>]*>/gi, '');
    
    return xmlString;
  }
}

/**
 * ENBD SOAP API Security
 */
class ENBDSOAPSecurity {
  constructor() {
    this.xxeService = new XXEPreventionService();
  }
  
  async handleSOAPRequest(req, res) {
    const xmlBody = req.body;
    
    // Validate XML
    const validation = this.xxeService.validateXML(xmlBody);
    
    if (!validation.valid) {
      return res.status(400).json({
        error: validation.error
      });
    }
    
    // Sanitize XML
    const sanitizedXML = this.xxeService.sanitizeXML(xmlBody);
    
    // Parse safely
    try {
      const parsed = await this.xxeService.safeXML2JSParse(sanitizedXML);
      
      // Process SOAP request
      const response = await this.processSOAPRequest(parsed);
      
      res.set('Content-Type', 'text/xml');
      res.send(response);
      
    } catch (error) {
      res.status(400).json({
        error: 'Invalid SOAP request'
      });
    }
  }
  
  async processSOAPRequest(data) {
    return '<soap:Envelope></soap:Envelope>';
  }
}

module.exports = {
  XXEPreventionService,
  ENBDSOAPSecurity
};
```

**XXE Prevention:**

1. ✅ **Disable external entities**
2. ✅ **Disable DTD processing**
3. ✅ **Validate XML before parsing**
4. ✅ **Use secure parsers**
5. ✅ **Sanitize XML input**

---

### Q35. How do you implement secure deserialization?

**Answer:**

```javascript
/**
 * Secure Deserialization Service
 */
class SecureDeserializationService {
  /**
   * ❌ VULNERABLE: eval() or Function()
   */
  vulnerableDeserialize(data) {
    // NEVER DO THIS!
    return eval(data); // Arbitrary code execution!
  }
  
  /**
   * ✅ SAFE: JSON.parse with validation
   */
  safeJSONParse(jsonString, schema) {
    try {
      const data = JSON.parse(jsonString);
      
      // Validate against schema
      if (!this.validateSchema(data, schema)) {
        throw new Error('Invalid data structure');
      }
      
      return data;
    } catch (error) {
      throw new Error('Invalid JSON');
    }
  }
  
  /**
   * Validate data structure
   */
  validateSchema(data, schema) {
    for (const [key, type] of Object.entries(schema)) {
      if (typeof data[key] !== type) {
        return false;
      }
    }
    return true;
  }
  
  /**
   * ✅ SAFE: Whitelist approach
   */
  deserializeWithWhitelist(data, allowedClasses) {
    if (data.__type && !allowedClasses.includes(data.__type)) {
      throw new Error('Class not allowed');
    }
    
    return data;
  }
}

module.exports = {
  SecureDeserializationService
};
```

**Deserialization Security:**

1. ✅ **Never use eval()**
2. ✅ **Use JSON.parse() instead of custom parsers**
3. ✅ **Validate data structure**
4. ✅ **Whitelist allowed classes**
5. ✅ **Type checking**

---

**Summary Q31-Q35:**
- SQL injection prevention (parameterized queries) ✅
- Comprehensive input validation (Joi schemas) ✅
- Command injection & file upload security ✅
- XXE prevention (disable external entities) ✅
- Secure deserialization (no eval) ✅

Continuing with more security questions...
