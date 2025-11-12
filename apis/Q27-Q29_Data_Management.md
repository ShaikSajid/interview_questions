# APIs Interview Questions (Q27-Q29): Data Management

## Q27: How do you implement database sharding for horizontally scaling banking data?

**Answer:**

Database sharding distributes data across multiple database servers to handle massive transaction volumes and improve performance for banking systems.

### Sharding Implementation:

```javascript
// shard-manager.js
const { Pool } = require('pg');
const crypto = require('crypto');

/**
 * Database Shard Manager
 */
class ShardManager {
    constructor(shardConfig) {
        // shardConfig = [
        //   { id: 'shard1', host: 'db1.enbd.com', ranges: [0, 3333] },
        //   { id: 'shard2', host: 'db2.enbd.com', ranges: [3334, 6666] },
        //   { id: 'shard3', host: 'db3.enbd.com', ranges: [6667, 9999] }
        // ]
        this.shards = new Map();
        
        shardConfig.forEach(shard => {
            this.shards.set(shard.id, {
                ...shard,
                pool: new Pool({
                    host: shard.host,
                    port: 5432,
                    database: 'enbd_banking',
                    user: process.env.DB_USER,
                    password: process.env.DB_PASSWORD,
                    max: 20,
                    min: 5,
                    idleTimeoutMillis: 30000
                })
            });
        });
    }

    /**
     * Hash-based sharding (consistent hashing)
     */
    getShardByHash(key) {
        const hash = crypto.createHash('md5').update(String(key)).digest('hex');
        const hashInt = parseInt(hash.substring(0, 8), 16);
        const shardKey = hashInt % 10000; // 0-9999 range
        
        for (const [shardId, shard] of this.shards) {
            const [min, max] = shard.ranges;
            if (shardKey >= min && shardKey <= max) {
                return shard;
            }
        }
        
        throw new Error(`No shard found for hash: ${shardKey}`);
    }

    /**
     * Range-based sharding (by customer ID)
     */
    getShardByRange(customerId) {
        // Extract numeric part from customer ID
        // e.g., CUST123456 -> 123456
        const numericId = parseInt(customerId.replace(/\D/g, ''));
        
        for (const [shardId, shard] of this.shards) {
            const [min, max] = shard.ranges;
            if (numericId >= min && numericId <= max) {
                return shard;
            }
        }
        
        throw new Error(`No shard found for customer ID: ${customerId}`);
    }

    /**
     * Geography-based sharding
     */
    getShardByGeography(country) {
        const geoShardMapping = {
            'UAE': 'shard1',
            'KSA': 'shard2',
            'EGYPT': 'shard3'
        };
        
        const shardId = geoShardMapping[country];
        if (!shardId) {
            throw new Error(`No shard for country: ${country}`);
        }
        
        return this.shards.get(shardId);
    }

    /**
     * Execute query on specific shard
     */
    async query(shardId, sql, params) {
        const shard = this.shards.get(shardId);
        if (!shard) {
            throw new Error(`Shard not found: ${shardId}`);
        }
        
        try {
            const result = await shard.pool.query(sql, params);
            return result.rows;
        } catch (error) {
            console.error(`Query failed on shard ${shardId}:`, error);
            throw error;
        }
    }

    /**
     * Execute query across all shards (scatter-gather)
     */
    async queryAllShards(sql, params) {
        const promises = Array.from(this.shards.entries()).map(
            async ([shardId, shard]) => {
                try {
                    const result = await shard.pool.query(sql, params);
                    return { shardId, data: result.rows };
                } catch (error) {
                    console.error(`Query failed on shard ${shardId}:`, error);
                    return { shardId, data: [], error: error.message };
                }
            }
        );
        
        const results = await Promise.all(promises);
        
        // Combine results from all shards
        return results.reduce((acc, { shardId, data }) => {
            return [...acc, ...data];
        }, []);
    }

    /**
     * Close all connections
     */
    async closeAll() {
        const promises = Array.from(this.shards.values()).map(
            shard => shard.pool.end()
        );
        await Promise.all(promises);
    }
}

/**
 * Customer Service with Sharding
 */
class ShardedCustomerService {
    constructor(shardManager) {
        this.shardManager = shardManager;
    }

    /**
     * Create customer (automatically sharded)
     */
    async createCustomer(customerData) {
        const { name, email, country, phoneNumber } = customerData;
        
        // Generate customer ID
        const customerId = this.generateCustomerId();
        
        // Determine shard based on hash
        const shard = this.shardManager.getShardByHash(customerId);
        
        const sql = `
            INSERT INTO customers (id, name, email, country, phone_number, created_at)
            VALUES ($1, $2, $3, $4, $5, NOW())
            RETURNING *
        `;
        
        const result = await shard.pool.query(sql, [
            customerId, name, email, country, phoneNumber
        ]);
        
        console.log(`Customer ${customerId} created on shard: ${shard.id}`);
        
        return result.rows[0];
    }

    /**
     * Get customer by ID (single shard lookup)
     */
    async getCustomer(customerId) {
        const shard = this.shardManager.getShardByHash(customerId);
        
        const sql = 'SELECT * FROM customers WHERE id = $1';
        const result = await shard.pool.query(sql, [customerId]);
        
        if (result.rows.length === 0) {
            throw new Error(`Customer not found: ${customerId}`);
        }
        
        return result.rows[0];
    }

    /**
     * Search customers by email (scatter-gather across all shards)
     */
    async searchCustomerByEmail(email) {
        const sql = 'SELECT * FROM customers WHERE email = $1 LIMIT 1';
        const results = await this.shardManager.queryAllShards(sql, [email]);
        
        if (results.length === 0) {
            throw new Error(`Customer not found with email: ${email}`);
        }
        
        return results[0];
    }

    /**
     * Get customers by country (geography-based)
     */
    async getCustomersByCountry(country, limit = 100) {
        const shard = this.shardManager.getShardByGeography(country);
        
        const sql = `
            SELECT * FROM customers 
            WHERE country = $1 
            ORDER BY created_at DESC 
            LIMIT $2
        `;
        
        const result = await shard.pool.query(sql, [country, limit]);
        return result.rows;
    }

    /**
     * Get total customer count (aggregate from all shards)
     */
    async getTotalCustomerCount() {
        const sql = 'SELECT COUNT(*) as count FROM customers';
        const results = await this.shardManager.queryAllShards(sql);
        
        const totalCount = results.reduce((sum, row) => {
            return sum + parseInt(row.count || 0);
        }, 0);
        
        return totalCount;
    }

    /**
     * Update customer (single shard)
     */
    async updateCustomer(customerId, updates) {
        const shard = this.shardManager.getShardByHash(customerId);
        
        const fields = Object.keys(updates);
        const values = Object.values(updates);
        
        const setClause = fields.map((field, index) => 
            `${field} = $${index + 2}`
        ).join(', ');
        
        const sql = `
            UPDATE customers 
            SET ${setClause}, updated_at = NOW()
            WHERE id = $1
            RETURNING *
        `;
        
        const result = await shard.pool.query(sql, [customerId, ...values]);
        
        if (result.rows.length === 0) {
            throw new Error(`Customer not found: ${customerId}`);
        }
        
        return result.rows[0];
    }

    generateCustomerId() {
        const timestamp = Date.now().toString(36);
        const random = crypto.randomBytes(4).toString('hex');
        return `CUST${timestamp}${random}`.toUpperCase();
    }
}

/**
 * Transaction Service with Sharding
 */
class ShardedTransactionService {
    constructor(shardManager) {
        this.shardManager = shardManager;
    }

    /**
     * Create transaction (same shard as account)
     */
    async createTransaction(accountId, transactionData) {
        const shard = this.shardManager.getShardByHash(accountId);
        
        const { type, amount, description } = transactionData;
        const transactionId = this.generateTransactionId();
        
        const client = await shard.pool.connect();
        
        try {
            await client.query('BEGIN');
            
            // Insert transaction
            const insertSql = `
                INSERT INTO transactions (
                    id, account_id, type, amount, description, status, created_at
                )
                VALUES ($1, $2, $3, $4, $5, 'completed', NOW())
                RETURNING *
            `;
            
            const txnResult = await client.query(insertSql, [
                transactionId, accountId, type, amount, description
            ]);
            
            // Update account balance
            const balanceChange = type === 'credit' ? amount : -amount;
            const updateSql = `
                UPDATE accounts 
                SET balance = balance + $1, updated_at = NOW()
                WHERE id = $2
                RETURNING balance
            `;
            
            const balanceResult = await client.query(updateSql, [balanceChange, accountId]);
            
            await client.query('COMMIT');
            
            return {
                transaction: txnResult.rows[0],
                newBalance: balanceResult.rows[0].balance
            };
            
        } catch (error) {
            await client.query('ROLLBACK');
            throw error;
        } finally {
            client.release();
        }
    }

    /**
     * Get account transactions (single shard)
     */
    async getAccountTransactions(accountId, options = {}) {
        const { limit = 50, offset = 0, startDate, endDate } = options;
        
        const shard = this.shardManager.getShardByHash(accountId);
        
        let sql = `
            SELECT * FROM transactions 
            WHERE account_id = $1
        `;
        const params = [accountId];
        let paramIndex = 2;
        
        if (startDate) {
            sql += ` AND created_at >= $${paramIndex}`;
            params.push(startDate);
            paramIndex++;
        }
        
        if (endDate) {
            sql += ` AND created_at <= $${paramIndex}`;
            params.push(endDate);
            paramIndex++;
        }
        
        sql += ` ORDER BY created_at DESC LIMIT $${paramIndex} OFFSET $${paramIndex + 1}`;
        params.push(limit, offset);
        
        const result = await shard.pool.query(sql, params);
        return result.rows;
    }

    /**
     * Get daily transaction summary (across all shards)
     */
    async getDailySummary(date) {
        const sql = `
            SELECT 
                COUNT(*) as transaction_count,
                SUM(CASE WHEN type = 'credit' THEN amount ELSE 0 END) as total_credits,
                SUM(CASE WHEN type = 'debit' THEN amount ELSE 0 END) as total_debits
            FROM transactions
            WHERE DATE(created_at) = $1
        `;
        
        const results = await this.shardManager.queryAllShards(sql, [date]);
        
        // Aggregate results
        return results.reduce((summary, row) => {
            return {
                transactionCount: summary.transactionCount + parseInt(row.transaction_count || 0),
                totalCredits: summary.totalCredits + parseFloat(row.total_credits || 0),
                totalDebits: summary.totalDebits + parseFloat(row.total_debits || 0)
            };
        }, { transactionCount: 0, totalCredits: 0, totalDebits: 0 });
    }

    generateTransactionId() {
        const timestamp = Date.now().toString(36);
        const random = crypto.randomBytes(6).toString('hex');
        return `TXN${timestamp}${random}`.toUpperCase();
    }
}

/**
 * Shard Rebalancing Service
 */
class ShardRebalancer {
    constructor(shardManager) {
        this.shardManager = shardManager;
    }

    /**
     * Analyze shard distribution
     */
    async analyzeShardDistribution() {
        const sql = `
            SELECT 
                COUNT(*) as record_count,
                pg_database_size(current_database()) as database_size
        `;
        
        const results = await this.shardManager.queryAllShards(sql);
        
        return results.map((result, index) => ({
            shardId: `shard${index + 1}`,
            recordCount: parseInt(result.record_count),
            databaseSize: result.database_size
        }));
    }

    /**
     * Migrate data between shards
     */
    async migrateData(fromShardId, toShardId, customerId) {
        const fromShard = this.shardManager.shards.get(fromShardId);
        const toShard = this.shardManager.shards.get(toShardId);
        
        if (!fromShard || !toShard) {
            throw new Error('Invalid shard IDs');
        }
        
        const fromClient = await fromShard.pool.connect();
        const toClient = await toShard.pool.connect();
        
        try {
            await fromClient.query('BEGIN');
            await toClient.query('BEGIN');
            
            // Copy customer data
            const customerResult = await fromClient.query(
                'SELECT * FROM customers WHERE id = $1',
                [customerId]
            );
            
            if (customerResult.rows.length === 0) {
                throw new Error(`Customer not found: ${customerId}`);
            }
            
            const customer = customerResult.rows[0];
            
            await toClient.query(
                `INSERT INTO customers (id, name, email, country, phone_number, created_at)
                 VALUES ($1, $2, $3, $4, $5, $6)`,
                [customer.id, customer.name, customer.email, customer.country, 
                 customer.phone_number, customer.created_at]
            );
            
            // Copy accounts
            const accountsResult = await fromClient.query(
                'SELECT * FROM accounts WHERE customer_id = $1',
                [customerId]
            );
            
            for (const account of accountsResult.rows) {
                await toClient.query(
                    `INSERT INTO accounts (id, customer_id, account_number, balance, created_at)
                     VALUES ($1, $2, $3, $4, $5)`,
                    [account.id, account.customer_id, account.account_number, 
                     account.balance, account.created_at]
                );
            }
            
            // Delete from source shard
            await fromClient.query('DELETE FROM accounts WHERE customer_id = $1', [customerId]);
            await fromClient.query('DELETE FROM customers WHERE id = $1', [customerId]);
            
            await fromClient.query('COMMIT');
            await toClient.query('COMMIT');
            
            console.log(`Migrated customer ${customerId} from ${fromShardId} to ${toShardId}`);
            
        } catch (error) {
            await fromClient.query('ROLLBACK');
            await toClient.query('ROLLBACK');
            throw error;
        } finally {
            fromClient.release();
            toClient.release();
        }
    }
}

// Usage Example
const shardConfig = [
    { id: 'shard1', host: 'db1.emiratesnbd.com', ranges: [0, 3333] },
    { id: 'shard2', host: 'db2.emiratesnbd.com', ranges: [3334, 6666] },
    { id: 'shard3', host: 'db3.emiratesnbd.com', ranges: [6667, 9999] }
];

const shardManager = new ShardManager(shardConfig);
const customerService = new ShardedCustomerService(shardManager);
const transactionService = new ShardedTransactionService(shardManager);

module.exports = {
    ShardManager,
    ShardedCustomerService,
    ShardedTransactionService,
    ShardRebalancer
};
```

---

## Q28: How do you implement zero-downtime database migrations for production banking APIs?

**Answer:**

Zero-downtime migrations ensure continuous API availability during schema changes, critical for 24/7 banking operations.

### Migration Framework:

```javascript
// migration-manager.js
const { Pool } = require('pg');
const fs = require('fs').promises;
const path = require('path');

/**
 * Database Migration Manager
 */
class MigrationManager {
    constructor(config) {
        this.pool = new Pool(config);
        this.migrationsDir = './migrations';
    }

    /**
     * Initialize migrations table
     */
    async initialize() {
        await this.pool.query(`
            CREATE TABLE IF NOT EXISTS schema_migrations (
                id SERIAL PRIMARY KEY,
                version VARCHAR(255) UNIQUE NOT NULL,
                name VARCHAR(255) NOT NULL,
                applied_at TIMESTAMP DEFAULT NOW(),
                execution_time_ms INTEGER,
                checksum VARCHAR(64)
            );
            
            CREATE INDEX IF NOT EXISTS idx_migrations_version 
            ON schema_migrations(version);
        `);
    }

    /**
     * Get applied migrations
     */
    async getAppliedMigrations() {
        const result = await this.pool.query(
            'SELECT version FROM schema_migrations ORDER BY version'
        );
        return result.rows.map(row => row.version);
    }

    /**
     * Get pending migrations
     */
    async getPendingMigrations() {
        const appliedMigrations = await this.getAppliedMigrations();
        const allMigrations = await this.getAllMigrationFiles();
        
        return allMigrations.filter(
            migration => !appliedMigrations.includes(migration.version)
        );
    }

    /**
     * Get all migration files
     */
    async getAllMigrationFiles() {
        const files = await fs.readdir(this.migrationsDir);
        
        const migrations = files
            .filter(file => file.endsWith('.sql'))
            .map(file => {
                const match = file.match(/^(\d+)_(.+)\.sql$/);
                if (!match) return null;
                
                return {
                    version: match[1],
                    name: match[2],
                    filename: file,
                    path: path.join(this.migrationsDir, file)
                };
            })
            .filter(Boolean)
            .sort((a, b) => a.version.localeCompare(b.version));
        
        return migrations;
    }

    /**
     * Run migrations
     */
    async migrate() {
        await this.initialize();
        
        const pendingMigrations = await this.getPendingMigrations();
        
        if (pendingMigrations.length === 0) {
            console.log('✅ No pending migrations');
            return { applied: 0 };
        }
        
        console.log(`📦 Found ${pendingMigrations.length} pending migrations`);
        
        for (const migration of pendingMigrations) {
            await this.applyMigration(migration);
        }
        
        console.log(`✅ Applied ${pendingMigrations.length} migrations`);
        
        return { applied: pendingMigrations.length };
    }

    /**
     * Apply single migration
     */
    async applyMigration(migration) {
        console.log(`⏳ Applying migration: ${migration.version}_${migration.name}`);
        
        const sql = await fs.readFile(migration.path, 'utf8');
        const checksum = this.calculateChecksum(sql);
        
        const client = await this.pool.connect();
        const startTime = Date.now();
        
        try {
            await client.query('BEGIN');
            
            // Execute migration
            await client.query(sql);
            
            // Record migration
            await client.query(
                `INSERT INTO schema_migrations (version, name, execution_time_ms, checksum)
                 VALUES ($1, $2, $3, $4)`,
                [migration.version, migration.name, Date.now() - startTime, checksum]
            );
            
            await client.query('COMMIT');
            
            console.log(`✅ Applied migration: ${migration.version} (${Date.now() - startTime}ms)`);
            
        } catch (error) {
            await client.query('ROLLBACK');
            console.error(`❌ Failed to apply migration ${migration.version}:`, error.message);
            throw error;
        } finally {
            client.release();
        }
    }

    /**
     * Rollback last migration
     */
    async rollback() {
        const result = await this.pool.query(
            'SELECT version, name FROM schema_migrations ORDER BY version DESC LIMIT 1'
        );
        
        if (result.rows.length === 0) {
            console.log('No migrations to rollback');
            return;
        }
        
        const { version, name } = result.rows[0];
        
        console.log(`🔄 Rolling back migration: ${version}_${name}`);
        
        // Look for down migration file
        const downFile = path.join(this.migrationsDir, `${version}_${name}_down.sql`);
        
        try {
            const sql = await fs.readFile(downFile, 'utf8');
            
            const client = await this.pool.connect();
            
            try {
                await client.query('BEGIN');
                await client.query(sql);
                await client.query('DELETE FROM schema_migrations WHERE version = $1', [version]);
                await client.query('COMMIT');
                
                console.log(`✅ Rolled back migration: ${version}`);
                
            } catch (error) {
                await client.query('ROLLBACK');
                throw error;
            } finally {
                client.release();
            }
            
        } catch (error) {
            console.error(`❌ Rollback failed: ${error.message}`);
            throw error;
        }
    }

    calculateChecksum(content) {
        const crypto = require('crypto');
        return crypto.createHash('sha256').update(content).digest('hex');
    }

    async close() {
        await this.pool.end();
    }
}

/**
 * Zero-Downtime Migration Strategy
 */
class ZeroDowntimeMigrator {
    /**
     * Example: Adding a new column with default value
     * Safe approach:
     * 1. Add column as nullable
     * 2. Backfill data
     * 3. Add NOT NULL constraint
     */
    static addColumnMigration() {
        return {
            up: `
                -- Step 1: Add nullable column
                ALTER TABLE customers 
                ADD COLUMN IF NOT EXISTS middle_name VARCHAR(100);
                
                -- Step 2: Add default value for new records
                ALTER TABLE customers 
                ALTER COLUMN middle_name SET DEFAULT '';
                
                -- Step 3: Backfill existing records (done in batches)
                -- This will be handled by application code
            `,
            
            backfill: async (pool) => {
                let offset = 0;
                const batchSize = 1000;
                
                while (true) {
                    const result = await pool.query(
                        `UPDATE customers 
                         SET middle_name = '' 
                         WHERE middle_name IS NULL 
                         LIMIT ${batchSize}
                         RETURNING id`
                    );
                    
                    if (result.rowCount === 0) break;
                    
                    console.log(`Backfilled ${result.rowCount} records`);
                    
                    // Sleep to avoid overwhelming the database
                    await new Promise(resolve => setTimeout(resolve, 100));
                }
            },
            
            addConstraint: `
                -- Step 4: Add NOT NULL constraint (after backfill complete)
                ALTER TABLE customers 
                ALTER COLUMN middle_name SET NOT NULL;
            `
        };
    }

    /**
     * Example: Renaming a column
     * Safe approach:
     * 1. Add new column
     * 2. Dual write to both columns
     * 3. Backfill old data
     * 4. Update application to read from new column
     * 5. Remove old column
     */
    static renameColumnMigration() {
        return {
            step1_add_new_column: `
                ALTER TABLE accounts 
                ADD COLUMN IF NOT EXISTS account_status VARCHAR(20);
                
                CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_accounts_account_status 
                ON accounts(account_status);
            `,
            
            step2_dual_write: `
                -- Trigger to sync old and new columns
                CREATE OR REPLACE FUNCTION sync_account_status()
                RETURNS TRIGGER AS $$
                BEGIN
                    IF NEW.status IS NOT NULL THEN
                        NEW.account_status := NEW.status;
                    END IF;
                    IF NEW.account_status IS NOT NULL THEN
                        NEW.status := NEW.account_status;
                    END IF;
                    RETURN NEW;
                END;
                $$ LANGUAGE plpgsql;
                
                DROP TRIGGER IF EXISTS sync_account_status_trigger ON accounts;
                CREATE TRIGGER sync_account_status_trigger
                BEFORE INSERT OR UPDATE ON accounts
                FOR EACH ROW EXECUTE FUNCTION sync_account_status();
            `,
            
            step3_backfill: `
                UPDATE accounts 
                SET account_status = status 
                WHERE account_status IS NULL;
            `,
            
            step4_remove_old_column: `
                -- After application is updated to use new column
                DROP TRIGGER IF EXISTS sync_account_status_trigger ON accounts;
                DROP FUNCTION IF EXISTS sync_account_status();
                ALTER TABLE accounts DROP COLUMN IF EXISTS status;
            `
        };
    }

    /**
     * Example: Adding an index without blocking writes
     */
    static addIndexMigration() {
        return `
            -- CREATE INDEX CONCURRENTLY doesn't block writes
            CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_created_at 
            ON transactions(created_at DESC);
            
            -- Analyze table after index creation
            ANALYZE transactions;
        `;
    }

    /**
     * Example: Modifying column type
     * Safe approach:
     * 1. Add new column with new type
     * 2. Backfill and transform data
     * 3. Switch application to use new column
     * 4. Remove old column
     */
    static changeColumnTypeMigration() {
        return {
            step1: `
                -- Change balance from DECIMAL(10,2) to DECIMAL(18,2)
                ALTER TABLE accounts 
                ADD COLUMN balance_new DECIMAL(18,2);
            `,
            
            step2: `
                -- Copy data
                UPDATE accounts 
                SET balance_new = balance 
                WHERE balance_new IS NULL;
            `,
            
            step3: `
                -- After application update
                ALTER TABLE accounts DROP COLUMN balance;
                ALTER TABLE accounts RENAME COLUMN balance_new TO balance;
                
                -- Add constraint
                ALTER TABLE accounts 
                ALTER COLUMN balance SET NOT NULL;
                
                -- Add index
                CREATE INDEX CONCURRENTLY idx_accounts_balance 
                ON accounts(balance);
            `
        };
    }
}

// Migration files examples:

// migrations/001_create_customers_table.sql
const migration001 = `
-- Create customers table
CREATE TABLE IF NOT EXISTS customers (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    country VARCHAR(3) NOT NULL,
    phone_number VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_country ON customers(country);
CREATE INDEX idx_customers_created_at ON customers(created_at DESC);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_customers_updated_at
BEFORE UPDATE ON customers
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
`;

// migrations/002_create_accounts_table.sql
const migration002 = `
-- Create accounts table
CREATE TABLE IF NOT EXISTS accounts (
    id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    account_number VARCHAR(30) UNIQUE NOT NULL,
    account_type VARCHAR(20) NOT NULL CHECK (account_type IN ('savings', 'current', 'fixed_deposit')),
    balance DECIMAL(18,2) DEFAULT 0 CHECK (balance >= 0),
    currency VARCHAR(3) DEFAULT 'AED',
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'frozen', 'closed')),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_accounts_customer_id ON accounts(customer_id);
CREATE INDEX idx_accounts_account_number ON accounts(account_number);
CREATE INDEX idx_accounts_status ON accounts(status) WHERE status = 'active';

CREATE TRIGGER update_accounts_updated_at
BEFORE UPDATE ON accounts
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
`;

// migrations/003_add_customer_kyc_fields.sql
const migration003 = `
-- Add KYC fields to customers (zero-downtime approach)
-- Step 1: Add nullable columns
ALTER TABLE customers 
ADD COLUMN IF NOT EXISTS kyc_status VARCHAR(20),
ADD COLUMN IF NOT EXISTS kyc_verified_at TIMESTAMP,
ADD COLUMN IF NOT EXISTS national_id VARCHAR(50);

-- Step 2: Add default for new records
ALTER TABLE customers 
ALTER COLUMN kyc_status SET DEFAULT 'pending';

-- Step 3: Create index concurrently
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_customers_kyc_status 
ON customers(kyc_status);

-- Step 4: Backfill existing records (in batches via application)
-- UPDATE customers SET kyc_status = 'pending' WHERE kyc_status IS NULL;
`;

module.exports = {
    MigrationManager,
    ZeroDowntimeMigrator
};
```

---

## Q29: How do you implement comprehensive backup and disaster recovery strategies for banking databases?

**Answer:**

Robust backup and recovery strategies ensure data integrity and business continuity for critical banking systems.

### Backup & Recovery Implementation:

```javascript
// backup-manager.js
const { exec } = require('child_process');
const util = require('util');
const execPromise = util.promisify(exec);
const fs = require('fs').promises;
const path = require('path');
const { S3Client, PutObjectCommand, ListObjectsV2Command } = require('@aws-sdk/client-s3');
const { createReadStream } = require('fs');

/**
 * Database Backup Manager
 */
class DatabaseBackupManager {
    constructor(config) {
        this.config = {
            host: config.host || 'localhost',
            port: config.port || 5432,
            database: config.database,
            user: config.user,
            password: config.password,
            backupDir: config.backupDir || './backups',
            s3Bucket: config.s3Bucket,
            retentionDays: config.retentionDays || 30
        };
        
        this.s3Client = new S3Client({ region: 'me-south-1' });
    }

    /**
     * Full database backup using pg_dump
     */
    async createFullBackup() {
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const filename = `enbd_banking_full_${timestamp}.sql.gz`;
        const filepath = path.join(this.config.backupDir, filename);
        
        console.log(`📦 Creating full backup: ${filename}`);
        
        const startTime = Date.now();
        
        try {
            // Ensure backup directory exists
            await fs.mkdir(this.config.backupDir, { recursive: true });
            
            // pg_dump with compression
            const command = `
                PGPASSWORD="${this.config.password}" pg_dump \
                -h ${this.config.host} \
                -p ${this.config.port} \
                -U ${this.config.user} \
                -d ${this.config.database} \
                -F c \
                -b \
                -v \
                --file="${filepath}"
            `;
            
            await execPromise(command);
            
            const stats = await fs.stat(filepath);
            const duration = Date.now() - startTime;
            
            console.log(`✅ Full backup completed in ${duration}ms`);
            console.log(`📊 Backup size: ${(stats.size / 1024 / 1024).toFixed(2)} MB`);
            
            // Upload to S3
            await this.uploadToS3(filepath, filename);
            
            // Clean up old backups
            await this.cleanupOldBackups();
            
            return {
                filename,
                filepath,
                size: stats.size,
                duration,
                timestamp: new Date()
            };
            
        } catch (error) {
            console.error('❌ Backup failed:', error.message);
            throw error;
        }
    }

    /**
     * Incremental backup using WAL archiving
     */
    async setupWALArchiving() {
        console.log('⚙️ Setting up WAL archiving for continuous backup');
        
        const walArchiveDir = path.join(this.config.backupDir, 'wal_archive');
        await fs.mkdir(walArchiveDir, { recursive: true });
        
        const archiveCommand = `
            test ! -f ${walArchiveDir}/%f && 
            cp %p ${walArchiveDir}/%f && 
            aws s3 cp ${walArchiveDir}/%f s3://${this.config.s3Bucket}/wal_archive/%f
        `;
        
        console.log(`
            Add these settings to postgresql.conf:
            
            wal_level = replica
            archive_mode = on
            archive_command = '${archiveCommand}'
            archive_timeout = 300  # Archive every 5 minutes
            
            Then restart PostgreSQL:
            sudo systemctl restart postgresql
        `);
    }

    /**
     * Point-in-Time Recovery (PITR) backup
     */
    async createPITRBackup() {
        console.log('📸 Creating base backup for Point-in-Time Recovery');
        
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const backupLabel = `pitr_${timestamp}`;
        const backupDir = path.join(this.config.backupDir, backupLabel);
        
        try {
            await fs.mkdir(backupDir, { recursive: true });
            
            const command = `
                PGPASSWORD="${this.config.password}" pg_basebackup \
                -h ${this.config.host} \
                -p ${this.config.port} \
                -U ${this.config.user} \
                -D ${backupDir} \
                -F tar \
                -z \
                -P \
                -X stream
            `;
            
            await execPromise(command);
            
            console.log(`✅ PITR base backup created: ${backupLabel}`);
            
            // Upload to S3
            await this.uploadDirectoryToS3(backupDir, `pitr/${backupLabel}`);
            
            return { backupLabel, backupDir };
            
        } catch (error) {
            console.error('❌ PITR backup failed:', error.message);
            throw error;
        }
    }

    /**
     * Logical backup of specific tables
     */
    async backupSpecificTables(tables) {
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const filename = `enbd_banking_tables_${timestamp}.sql.gz`;
        const filepath = path.join(this.config.backupDir, filename);
        
        console.log(`📦 Backing up tables: ${tables.join(', ')}`);
        
        try {
            const tableArgs = tables.map(t => `-t ${t}`).join(' ');
            
            const command = `
                PGPASSWORD="${this.config.password}" pg_dump \
                -h ${this.config.host} \
                -p ${this.config.port} \
                -U ${this.config.user} \
                -d ${this.config.database} \
                ${tableArgs} \
                | gzip > "${filepath}"
            `;
            
            await execPromise(command);
            
            console.log(`✅ Table backup completed: ${filename}`);
            
            return { filename, filepath };
            
        } catch (error) {
            console.error('❌ Table backup failed:', error.message);
            throw error;
        }
    }

    /**
     * Restore from backup
     */
    async restoreFromBackup(backupFile) {
        console.log(`🔄 Restoring from backup: ${backupFile}`);
        
        try {
            // Drop and recreate database
            const dropCommand = `
                PGPASSWORD="${this.config.password}" psql \
                -h ${this.config.host} \
                -p ${this.config.port} \
                -U ${this.config.user} \
                -d postgres \
                -c "DROP DATABASE IF EXISTS ${this.config.database};"
            `;
            
            await execPromise(dropCommand);
            
            const createCommand = `
                PGPASSWORD="${this.config.password}" psql \
                -h ${this.config.host} \
                -p ${this.config.port} \
                -U ${this.config.user} \
                -d postgres \
                -c "CREATE DATABASE ${this.config.database};"
            `;
            
            await execPromise(createCommand);
            
            // Restore backup
            const restoreCommand = `
                PGPASSWORD="${this.config.password}" pg_restore \
                -h ${this.config.host} \
                -p ${this.config.port} \
                -U ${this.config.user} \
                -d ${this.config.database} \
                -v \
                "${backupFile}"
            `;
            
            await execPromise(restoreCommand);
            
            console.log('✅ Restore completed successfully');
            
        } catch (error) {
            console.error('❌ Restore failed:', error.message);
            throw error;
        }
    }

    /**
     * Point-in-Time Recovery
     */
    async performPITR(backupLabel, recoveryTargetTime) {
        console.log(`🔄 Performing Point-in-Time Recovery to: ${recoveryTargetTime}`);
        
        const recoveryConf = `
            restore_command = 'cp ${this.config.backupDir}/wal_archive/%f %p'
            recovery_target_time = '${recoveryTargetTime}'
            recovery_target_action = 'promote'
        `;
        
        console.log(`
            1. Stop PostgreSQL
            2. Replace data directory with base backup
            3. Create recovery.signal file
            4. Add to postgresql.auto.conf:
            
            ${recoveryConf}
            
            5. Start PostgreSQL
            6. Monitor logs for recovery completion
        `);
    }

    /**
     * Upload backup to S3
     */
    async uploadToS3(filepath, filename) {
        console.log(`☁️  Uploading to S3: ${filename}`);
        
        try {
            const fileStream = createReadStream(filepath);
            
            const command = new PutObjectCommand({
                Bucket: this.config.s3Bucket,
                Key: `backups/${filename}`,
                Body: fileStream,
                ServerSideEncryption: 'AES256',
                StorageClass: 'STANDARD_IA',
                Metadata: {
                    'backup-date': new Date().toISOString(),
                    'database': this.config.database
                }
            });
            
            await this.s3Client.send(command);
            
            console.log('✅ Upload to S3 completed');
            
        } catch (error) {
            console.error('❌ S3 upload failed:', error.message);
            throw error;
        }
    }

    /**
     * Upload directory to S3
     */
    async uploadDirectoryToS3(dirPath, s3Prefix) {
        const files = await fs.readdir(dirPath);
        
        for (const file of files) {
            const filepath = path.join(dirPath, file);
            const stats = await fs.stat(filepath);
            
            if (stats.isFile()) {
                await this.uploadToS3(filepath, `${s3Prefix}/${file}`);
            }
        }
    }

    /**
     * Clean up old backups
     */
    async cleanupOldBackups() {
        console.log('🧹 Cleaning up old backups...');
        
        try {
            // List S3 backups
            const listCommand = new ListObjectsV2Command({
                Bucket: this.config.s3Bucket,
                Prefix: 'backups/'
            });
            
            const { Contents } = await this.s3Client.send(listCommand);
            
            if (!Contents) return;
            
            const cutoffDate = new Date();
            cutoffDate.setDate(cutoffDate.getDate() - this.config.retentionDays);
            
            const oldBackups = Contents.filter(item => 
                new Date(item.LastModified) < cutoffDate
            );
            
            console.log(`Found ${oldBackups.length} backups older than ${this.config.retentionDays} days`);
            
            // Delete old backups (implementation omitted for brevity)
            
        } catch (error) {
            console.error('Cleanup failed:', error.message);
        }
    }

    /**
     * Verify backup integrity
     */
    async verifyBackup(backupFile) {
        console.log(`🔍 Verifying backup: ${backupFile}`);
        
        try {
            const command = `
                PGPASSWORD="${this.config.password}" pg_restore \
                --list "${backupFile}" > /dev/null
            `;
            
            await execPromise(command);
            
            console.log('✅ Backup verification successful');
            return { valid: true };
            
        } catch (error) {
            console.error('❌ Backup verification failed:', error.message);
            return { valid: false, error: error.message };
        }
    }

    /**
     * Schedule automated backups
     */
    scheduleBackups(cronSchedule = '0 2 * * *') {
        const cron = require('node-cron');
        
        console.log(`⏰ Scheduling daily backups at 2 AM (cron: ${cronSchedule})`);
        
        cron.schedule(cronSchedule, async () => {
            console.log('Starting scheduled backup...');
            
            try {
                await this.createFullBackup();
                console.log('Scheduled backup completed');
            } catch (error) {
                console.error('Scheduled backup failed:', error);
                // Send alert notification
            }
        });
    }
}

/**
 * Disaster Recovery Manager
 */
class DisasterRecoveryManager {
    constructor(primaryConfig, replicaConfigs) {
        this.primary = primaryConfig;
        this.replicas = replicaConfigs;
    }

    /**
     * Setup streaming replication
     */
    async setupStreamingReplication() {
        console.log('⚙️ Setting up streaming replication');
        
        const primaryConf = `
            # Primary server postgresql.conf
            wal_level = replica
            max_wal_senders = 10
            max_replication_slots = 10
            synchronous_commit = on
            synchronous_standby_names = 'replica1,replica2'
        `;
        
        const replicaConf = `
            # Replica server postgresql.conf
            hot_standby = on
            max_standby_streaming_delay = 30s
            wal_receiver_status_interval = 10s
            hot_standby_feedback = on
        `;
        
        const recoveryConf = `
            # Replica server postgresql.auto.conf
            primary_conninfo = 'host=${this.primary.host} port=5432 user=replicator password=...'
            primary_slot_name = 'replica1'
        `;
        
        console.log(`
            Primary Configuration:
            ${primaryConf}
            
            Replica Configuration:
            ${replicaConf}
            
            Recovery Configuration:
            ${recoveryConf}
        `);
    }

    /**
     * Failover to replica
     */
    async performFailover(replicaId) {
        console.log(`🔄 Failing over to replica: ${replicaId}`);
        
        const replica = this.replicas.find(r => r.id === replicaId);
        if (!replica) {
            throw new Error(`Replica not found: ${replicaId}`);
        }
        
        console.log(`
            1. Stop application traffic
            2. SSH to replica server
            3. Promote replica to primary:
               pg_ctl promote -D /var/lib/postgresql/data
            4. Update application connection string
            5. Resume application traffic
            6. Setup new replica from old primary
        `);
    }

    /**
     * Monitor replication lag
     */
    async monitorReplicationLag() {
        // Query on primary
        const lagQuery = `
            SELECT 
                client_addr,
                state,
                sent_lsn,
                write_lsn,
                replay_lsn,
                sync_state,
                pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
            FROM pg_stat_replication;
        `;
        
        console.log('Monitoring replication lag:', lagQuery);
    }
}

// Usage Example
const backupManager = new DatabaseBackupManager({
    host: 'localhost',
    database: 'enbd_banking',
    user: 'postgres',
    password: process.env.DB_PASSWORD,
    backupDir: './backups',
    s3Bucket: 'enbd-db-backups',
    retentionDays: 30
});

// Schedule daily backups
backupManager.scheduleBackups('0 2 * * *');

module.exports = {
    DatabaseBackupManager,
    DisasterRecoveryManager
};
```

---
