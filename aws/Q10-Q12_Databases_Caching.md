# AWS Interview Questions: Databases & Caching (Q10-Q12)

## Question 10: Explain Amazon DynamoDB and its key features

### 📋 Answer

**Amazon DynamoDB** is a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability.

### Key Features:

1. **Fully Managed**: No servers to provision or manage
2. **Performance at Scale**: Single-digit millisecond latency
3. **Flexible Schema**: Store any type of data
4. **Auto Scaling**: Automatically adjusts capacity
5. **Built-in Security**: Encryption at rest and in transit
6. **Global Tables**: Multi-region, multi-master replication
7. **Streams**: Capture data changes in real-time
8. **ACID Transactions**: Support for complex operations

### DynamoDB Structure:

```
DynamoDB Table
├── Primary Key
│   ├── Partition Key (Required)
│   └── Sort Key (Optional)
├── Attributes (Schema-less)
├── Indexes
│   ├── Local Secondary Index (LSI)
│   └── Global Secondary Index (GSI)
└── Streams (Change Data Capture)
```

### Complete DynamoDB Implementation:

```javascript
// dynamodb-manager.js
import {
  DynamoDBClient,
  CreateTableCommand,
  DescribeTableCommand,
  UpdateTableCommand,
  DeleteTableCommand,
  ListTablesCommand
} from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient,
  PutCommand,
  GetCommand,
  UpdateCommand,
  DeleteCommand,
  QueryCommand,
  ScanCommand,
  BatchWriteCommand,
  BatchGetCommand,
  TransactWriteCommand
} from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({ region: 'us-east-1' });
const docClient = DynamoDBDocumentClient.from(client);

// 1. Create Table
async function createTable(tableName) {
  const command = new CreateTableCommand({
    TableName: tableName,
    KeySchema: [
      { AttributeName: 'userId', KeyType: 'HASH' },  // Partition key
      { AttributeName: 'timestamp', KeyType: 'RANGE' }  // Sort key
    ],
    AttributeDefinitions: [
      { AttributeName: 'userId', AttributeType: 'S' },
      { AttributeName: 'timestamp', AttributeType: 'N' },
      { AttributeName: 'status', AttributeType: 'S' },
      { AttributeName: 'email', AttributeType: 'S' }
    ],
    BillingMode: 'PAY_PER_REQUEST',  // or 'PROVISIONED'
    GlobalSecondaryIndexes: [
      {
        IndexName: 'EmailIndex',
        KeySchema: [
          { AttributeName: 'email', KeyType: 'HASH' }
        ],
        Projection: {
          ProjectionType: 'ALL'
        }
      },
      {
        IndexName: 'StatusIndex',
        KeySchema: [
          { AttributeName: 'status', KeyType: 'HASH' },
          { AttributeName: 'timestamp', KeyType: 'RANGE' }
        ],
        Projection: {
          ProjectionType: 'ALL'
        }
      }
    ],
    StreamSpecification: {
      StreamEnabled: true,
      StreamViewType: 'NEW_AND_OLD_IMAGES'
    },
    Tags: [
      { Key: 'Environment', Value: 'Production' },
      { Key: 'Application', Value: 'UserManagement' }
    ]
  });
  
  try {
    const response = await client.send(command);
    console.log('Table created:', response.TableDescription.TableName);
    return response.TableDescription;
  } catch (error) {
    console.error('Failed to create table:', error);
    throw error;
  }
}

// 2. Put Item
async function putItem(tableName, item) {
  const command = new PutCommand({
    TableName: tableName,
    Item: item,
    // Conditional expression to prevent overwrites
    ConditionExpression: 'attribute_not_exists(userId)'
  });
  
  try {
    await docClient.send(command);
    console.log('Item added successfully');
    return item;
  } catch (error) {
    if (error.name === 'ConditionalCheckFailedException') {
      console.error('Item already exists');
    } else {
      console.error('Failed to put item:', error);
    }
    throw error;
  }
}

// 3. Get Item
async function getItem(tableName, key) {
  const command = new GetCommand({
    TableName: tableName,
    Key: key,
    ConsistentRead: true  // Strong consistency
  });
  
  try {
    const response = await docClient.send(command);
    return response.Item;
  } catch (error) {
    console.error('Failed to get item:', error);
    throw error;
  }
}

// 4. Update Item
async function updateItem(tableName, key, updates) {
  // Build update expression dynamically
  const updateExpressions = [];
  const expressionAttributeNames = {};
  const expressionAttributeValues = {};
  
  Object.keys(updates).forEach((key, index) => {
    const attrName = `#attr${index}`;
    const attrValue = `:val${index}`;
    
    updateExpressions.push(`${attrName} = ${attrValue}`);
    expressionAttributeNames[attrName] = key;
    expressionAttributeValues[attrValue] = updates[key];
  });
  
  const command = new UpdateCommand({
    TableName: tableName,
    Key: key,
    UpdateExpression: `SET ${updateExpressions.join(', ')}`,
    ExpressionAttributeNames: expressionAttributeNames,
    ExpressionAttributeValues: expressionAttributeValues,
    ReturnValues: 'ALL_NEW'
  });
  
  try {
    const response = await docClient.send(command);
    console.log('Item updated');
    return response.Attributes;
  } catch (error) {
    console.error('Failed to update item:', error);
    throw error;
  }
}

// 5. Delete Item
async function deleteItem(tableName, key) {
  const command = new DeleteCommand({
    TableName: tableName,
    Key: key,
    ReturnValues: 'ALL_OLD'
  });
  
  try {
    const response = await docClient.send(command);
    console.log('Item deleted');
    return response.Attributes;
  } catch (error) {
    console.error('Failed to delete item:', error);
    throw error;
  }
}

// 6. Query (Efficient - uses indexes)
async function queryByUserId(tableName, userId, statusFilter = null) {
  const params = {
    TableName: tableName,
    KeyConditionExpression: 'userId = :userId',
    ExpressionAttributeValues: {
      ':userId': userId
    },
    ScanIndexForward: false,  // Sort descending
    Limit: 100
  };
  
  // Add filter if provided
  if (statusFilter) {
    params.FilterExpression = '#status = :status';
    params.ExpressionAttributeNames = { '#status': 'status' };
    params.ExpressionAttributeValues[':status'] = statusFilter;
  }
  
  const command = new QueryCommand(params);
  
  try {
    const response = await docClient.send(command);
    console.log(`Found ${response.Items.length} items`);
    return response.Items;
  } catch (error) {
    console.error('Query failed:', error);
    throw error;
  }
}

// 7. Query with GSI
async function queryByEmail(tableName, email) {
  const command = new QueryCommand({
    TableName: tableName,
    IndexName: 'EmailIndex',
    KeyConditionExpression: 'email = :email',
    ExpressionAttributeValues: {
      ':email': email
    }
  });
  
  const response = await docClient.send(command);
  return response.Items;
}

// 8. Scan (Full table scan - use sparingly)
async function scanTable(tableName, filters = {}) {
  const command = new ScanCommand({
    TableName: tableName,
    FilterExpression: filters.expression,
    ExpressionAttributeNames: filters.names,
    ExpressionAttributeValues: filters.values,
    Limit: 100
  });
  
  try {
    const response = await docClient.send(command);
    console.log(`Scanned ${response.Count} items`);
    return response.Items;
  } catch (error) {
    console.error('Scan failed:', error);
    throw error;
  }
}

// 9. Batch Write (Up to 25 items)
async function batchWriteItems(tableName, items) {
  const putRequests = items.map(item => ({
    PutRequest: { Item: item }
  }));
  
  const command = new BatchWriteCommand({
    RequestItems: {
      [tableName]: putRequests
    }
  });
  
  try {
    const response = await docClient.send(command);
    
    // Handle unprocessed items
    if (response.UnprocessedItems && Object.keys(response.UnprocessedItems).length > 0) {
      console.warn('Some items were not processed:', response.UnprocessedItems);
    } else {
      console.log(`Batch wrote ${items.length} items`);
    }
    
    return response;
  } catch (error) {
    console.error('Batch write failed:', error);
    throw error;
  }
}

// 10. Batch Get (Up to 100 items)
async function batchGetItems(tableName, keys) {
  const command = new BatchGetCommand({
    RequestItems: {
      [tableName]: {
        Keys: keys,
        ConsistentRead: true
      }
    }
  });
  
  try {
    const response = await docClient.send(command);
    return response.Responses[tableName];
  } catch (error) {
    console.error('Batch get failed:', error);
    throw error;
  }
}

// 11. Transactions (ACID)
async function performTransaction(tableName) {
  const command = new TransactWriteCommand({
    TransactItems: [
      {
        Put: {
          TableName: tableName,
          Item: {
            userId: 'user123',
            timestamp: Date.now(),
            action: 'deposit',
            amount: 100
          }
        }
      },
      {
        Update: {
          TableName: tableName,
          Key: { userId: 'user123', timestamp: 1234567890 },
          UpdateExpression: 'SET balance = balance + :amount',
          ExpressionAttributeValues: {
            ':amount': 100
          }
        }
      },
      {
        Delete: {
          TableName: tableName,
          Key: { userId: 'user123', timestamp: 9876543210 }
        }
      }
    ]
  });
  
  try {
    await docClient.send(command);
    console.log('Transaction completed successfully');
  } catch (error) {
    console.error('Transaction failed:', error);
    throw error;
  }
}
```

### Real-World Application Example:

```javascript
// user-service.js - User Management System
class UserService {
  constructor(tableName) {
    this.tableName = tableName;
  }
  
  // Create new user
  async createUser(userData) {
    const user = {
      userId: this.generateUserId(),
      timestamp: Date.now(),
      email: userData.email,
      name: userData.name,
      status: 'active',
      createdAt: new Date().toISOString(),
      profile: {
        bio: userData.bio || '',
        avatar: userData.avatar || null
      },
      settings: {
        notifications: true,
        theme: 'light'
      }
    };
    
    try {
      await putItem(this.tableName, user);
      console.log('User created:', user.userId);
      return user;
    } catch (error) {
      if (error.name === 'ConditionalCheckFailedException') {
        throw new Error('User already exists');
      }
      throw error;
    }
  }
  
  // Get user by ID
  async getUserById(userId) {
    const user = await getItem(this.tableName, {
      userId: userId,
      timestamp: await this.getLatestTimestamp(userId)
    });
    
    if (!user) {
      throw new Error('User not found');
    }
    
    return user;
  }
  
  // Get user by email (using GSI)
  async getUserByEmail(email) {
    const users = await queryByEmail(this.tableName, email);
    
    if (users.length === 0) {
      throw new Error('User not found');
    }
    
    return users[0];
  }
  
  // Update user profile
  async updateUserProfile(userId, updates) {
    const timestamp = await this.getLatestTimestamp(userId);
    
    const updatedUser = await updateItem(
      this.tableName,
      { userId, timestamp },
      {
        'profile.bio': updates.bio,
        'profile.avatar': updates.avatar,
        updatedAt: new Date().toISOString()
      }
    );
    
    return updatedUser;
  }
  
  // Deactivate user
  async deactivateUser(userId) {
    const timestamp = await this.getLatestTimestamp(userId);
    
    return await updateItem(
      this.tableName,
      { userId, timestamp },
      {
        status: 'inactive',
        deactivatedAt: new Date().toISOString()
      }
    );
  }
  
  // Get user activity history
  async getUserActivity(userId, limit = 50) {
    const activities = await queryByUserId(this.tableName, userId);
    return activities.slice(0, limit);
  }
  
  // Get active users
  async getActiveUsers() {
    return await scanTable(this.tableName, {
      expression: '#status = :status',
      names: { '#status': 'status' },
      values: { ':status': 'active' }
    });
  }
  
  // Bulk create users
  async bulkCreateUsers(usersData) {
    const users = usersData.map(userData => ({
      userId: this.generateUserId(),
      timestamp: Date.now(),
      email: userData.email,
      name: userData.name,
      status: 'active',
      createdAt: new Date().toISOString()
    }));
    
    // Process in batches of 25
    const batches = this.chunkArray(users, 25);
    
    for (const batch of batches) {
      await batchWriteItems(this.tableName, batch);
    }
    
    console.log(`Created ${users.length} users`);
    return users;
  }
  
  // Transfer credits between users (Transaction)
  async transferCredits(fromUserId, toUserId, amount) {
    const timestamp = Date.now();
    
    const command = new TransactWriteCommand({
      TransactItems: [
        {
          Update: {
            TableName: this.tableName,
            Key: { userId: fromUserId, timestamp: await this.getLatestTimestamp(fromUserId) },
            UpdateExpression: 'SET credits = credits - :amount',
            ConditionExpression: 'credits >= :amount',
            ExpressionAttributeValues: {
              ':amount': amount
            }
          }
        },
        {
          Update: {
            TableName: this.tableName,
            Key: { userId: toUserId, timestamp: await this.getLatestTimestamp(toUserId) },
            UpdateExpression: 'SET credits = credits + :amount',
            ExpressionAttributeValues: {
              ':amount': amount
            }
          }
        },
        {
          Put: {
            TableName: this.tableName,
            Item: {
              userId: fromUserId,
              timestamp: timestamp,
              type: 'transaction',
              action: 'transfer_out',
              amount: amount,
              toUser: toUserId,
              createdAt: new Date().toISOString()
            }
          }
        },
        {
          Put: {
            TableName: this.tableName,
            Item: {
              userId: toUserId,
              timestamp: timestamp,
              type: 'transaction',
              action: 'transfer_in',
              amount: amount,
              fromUser: fromUserId,
              createdAt: new Date().toISOString()
            }
          }
        }
      ]
    });
    
    try {
      await docClient.send(command);
      console.log(`Transferred ${amount} credits from ${fromUserId} to ${toUserId}`);
    } catch (error) {
      if (error.name === 'TransactionCanceledException') {
        throw new Error('Insufficient credits');
      }
      throw error;
    }
  }
  
  // Helper methods
  generateUserId() {
    return `user_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
  
  async getLatestTimestamp(userId) {
    const items = await queryByUserId(this.tableName, userId);
    return items[0]?.timestamp || Date.now();
  }
  
  chunkArray(array, size) {
    const chunks = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}

// Usage
const userService = new UserService('Users');

// Create user
const newUser = await userService.createUser({
  email: 'john@example.com',
  name: 'John Doe',
  bio: 'Software Engineer'
});

// Get user
const user = await userService.getUserById(newUser.userId);

// Update profile
await userService.updateUserProfile(newUser.userId, {
  bio: 'Senior Software Engineer',
  avatar: 'https://example.com/avatar.jpg'
});

// Transfer credits
await userService.transferCredits('user123', 'user456', 100);
```

### DynamoDB Streams Example:

```javascript
// streams-processor.js
import {
  DynamoDBStreamsClient,
  GetRecordsCommand,
  GetShardIteratorCommand
} from '@aws-sdk/client-dynamodb-streams';

const streamsClient = new DynamoDBStreamsClient({ region: 'us-east-1' });

async function processStreamRecords(streamArn) {
  // Get shard iterator
  const shardIteratorResponse = await streamsClient.send(
    new GetShardIteratorCommand({
      StreamArn: streamArn,
      ShardId: 'shardId-00000001234567890',
      ShardIteratorType: 'LATEST'
    })
  );
  
  let shardIterator = shardIteratorResponse.ShardIterator;
  
  // Poll for records
  while (true) {
    const recordsResponse = await streamsClient.send(
      new GetRecordsCommand({
        ShardIterator: shardIterator,
        Limit: 100
      })
    );
    
    // Process records
    for (const record of recordsResponse.Records) {
      console.log('Event:', record.eventName);
      console.log('New Image:', record.dynamodb.NewImage);
      console.log('Old Image:', record.dynamodb.OldImage);
      
      // Handle different event types
      switch (record.eventName) {
        case 'INSERT':
          await handleInsert(record.dynamodb.NewImage);
          break;
        case 'MODIFY':
          await handleModify(record.dynamodb.NewImage, record.dynamodb.OldImage);
          break;
        case 'REMOVE':
          await handleRemove(record.dynamodb.OldImage);
          break;
      }
    }
    
    shardIterator = recordsResponse.NextShardIterator;
    
    if (!shardIterator) break;
    
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}

async function handleInsert(newImage) {
  console.log('New item created:', newImage);
  // Send welcome email, update cache, etc.
}

async function handleModify(newImage, oldImage) {
  console.log('Item modified:', { old: oldImage, new: newImage });
  // Invalidate cache, trigger webhooks, etc.
}

async function handleRemove(oldImage) {
  console.log('Item deleted:', oldImage);
  // Clean up related resources, log audit trail, etc.
}
```

### Best Practices:

1. ✅ Design partition keys for even distribution
2. ✅ Use sparse indexes to reduce storage costs
3. ✅ Implement exponential backoff for retries
4. ✅ Use BatchWrite for bulk operations
5. ✅ Enable point-in-time recovery for critical tables
6. ✅ Use DynamoDB Streams for change notifications
7. ✅ Optimize with projection expressions
8. ✅ Use conditional writes to prevent race conditions
9. ✅ Monitor with CloudWatch metrics
10. ✅ Use DAX for read-heavy workloads

---

## Question 11: Explain Amazon ElastiCache (Redis/Memcached)

### 📋 Answer

**Amazon ElastiCache** is a fully managed in-memory data store service that supports Redis and Memcached engines.

### ElastiCache Engines:

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data Types** | Strings, Lists, Sets, Hashes, Sorted Sets | Strings only |
| **Persistence** | Yes (snapshots, AOF) | No |
| **Replication** | Yes (master-replica) | No (sharding only) |
| **Transactions** | Yes | No |
| **Pub/Sub** | Yes | No |
| **Multi-AZ** | Yes (automatic failover) | No |
| **Backup/Restore** | Yes | No |
| **Use Case** | Complex data structures, persistence | Simple caching |

### Complete Redis Implementation:

```javascript
// redis-cache-manager.js
import { createClient } from 'redis';

// Create Redis client
const redisClient = createClient({
  socket: {
    host: 'my-redis-cluster.abc123.0001.use1.cache.amazonaws.com',
    port: 6379
  },
  password: process.env.REDIS_PASSWORD
});

redisClient.on('error', (err) => console.error('Redis Client Error', err));
redisClient.on('connect', () => console.log('Connected to Redis'));
redisClient.on('reconnecting', () => console.log('Reconnecting to Redis'));

await redisClient.connect();

// Cache Manager Class
class CacheManager {
  constructor(client) {
    this.client = client;
    this.defaultTTL = 3600; // 1 hour
  }
  
  // 1. Set value with expiration
  async set(key, value, ttl = this.defaultTTL) {
    const serialized = JSON.stringify(value);
    await this.client.setEx(key, ttl, serialized);
    console.log(`Cached: ${key} (TTL: ${ttl}s)`);
  }
  
  // 2. Get value
  async get(key) {
    const value = await this.client.get(key);
    
    if (!value) {
      console.log(`Cache miss: ${key}`);
      return null;
    }
    
    console.log(`Cache hit: ${key}`);
    return JSON.parse(value);
  }
  
  // 3. Delete key
  async delete(key) {
    await this.client.del(key);
    console.log(`Deleted: ${key}`);
  }
  
  // 4. Check if key exists
  async exists(key) {
    const exists = await this.client.exists(key);
    return exists === 1;
  }
  
  // 5. Get multiple keys
  async mGet(keys) {
    const values = await this.client.mGet(keys);
    return values.map(v => v ? JSON.parse(v) : null);
  }
  
  // 6. Set multiple keys
  async mSet(keyValuePairs) {
    const pairs = [];
    for (const [key, value] of Object.entries(keyValuePairs)) {
      pairs.push(key, JSON.stringify(value));
    }
    await this.client.mSet(pairs);
    console.log(`Set ${Object.keys(keyValuePairs).length} keys`);
  }
  
  // 7. Increment counter
  async increment(key, amount = 1) {
    const newValue = await this.client.incrBy(key, amount);
    return newValue;
  }
  
  // 8. Decrement counter
  async decrement(key, amount = 1) {
    const newValue = await this.client.decrBy(key, amount);
    return newValue;
  }
  
  // 9. Set expiration on existing key
  async expire(key, ttl) {
    await this.client.expire(key, ttl);
  }
  
  // 10. Get time-to-live
  async ttl(key) {
    const ttl = await this.client.ttl(key);
    return ttl;
  }
  
  // 11. Clear all keys (use with caution!)
  async flushAll() {
    await this.client.flushAll();
    console.log('Cache cleared');
  }
  
  // 12. Get cache statistics
  async getStats() {
    const info = await this.client.info();
    return {
      uptime: info.uptime_in_seconds,
      connectedClients: info.connected_clients,
      usedMemory: info.used_memory_human,
      totalKeys: await this.client.dbSize()
    };
  }
}

// Usage
const cache = new CacheManager(redisClient);

// Basic caching
await cache.set('user:123', { name: 'John', email: 'john@example.com' }, 3600);
const user = await cache.get('user:123');

// Counter
await cache.increment('page:views:home');
await cache.increment('api:requests', 10);
```

### Advanced Redis Patterns:

```javascript
// advanced-redis-patterns.js

// 1. Cache-Aside Pattern
class UserRepository {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }
  
  async getUser(userId) {
    const cacheKey = `user:${userId}`;
    
    // Try cache first
    let user = await this.cache.get(cacheKey);
    
    if (user) {
      console.log('Cache hit');
      return user;
    }
    
    console.log('Cache miss - fetching from database');
    
    // Fetch from database
    user = await this.db.getUserById(userId);
    
    if (user) {
      // Store in cache for future requests
      await this.cache.set(cacheKey, user, 3600);
    }
    
    return user;
  }
  
  async updateUser(userId, updates) {
    // Update database
    const user = await this.db.updateUser(userId, updates);
    
    // Invalidate cache
    const cacheKey = `user:${userId}`;
    await this.cache.delete(cacheKey);
    
    return user;
  }
}

// 2. Write-Through Cache
class WriteThrough {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }
  
  async saveUser(user) {
    // Write to database
    await this.db.saveUser(user);
    
    // Write to cache
    await this.cache.set(`user:${user.id}`, user, 3600);
    
    return user;
  }
}

// 3. Session Store
class SessionManager {
  constructor(cache) {
    this.cache = cache;
    this.sessionTTL = 86400; // 24 hours
  }
  
  async createSession(userId, data) {
    const sessionId = this.generateSessionId();
    const sessionKey = `session:${sessionId}`;
    
    const session = {
      id: sessionId,
      userId: userId,
      data: data,
      createdAt: new Date().toISOString()
    };
    
    await this.cache.set(sessionKey, session, this.sessionTTL);
    
    return sessionId;
  }
  
  async getSession(sessionId) {
    const sessionKey = `session:${sessionId}`;
    const session = await this.cache.get(sessionKey);
    
    if (session) {
      // Refresh TTL on access
      await this.cache.expire(sessionKey, this.sessionTTL);
    }
    
    return session;
  }
  
  async destroySession(sessionId) {
    const sessionKey = `session:${sessionId}`;
    await this.cache.delete(sessionKey);
  }
  
  generateSessionId() {
    return `sess_${Date.now()}_${Math.random().toString(36).substr(2, 16)}`;
  }
}

// 4. Rate Limiting
class RateLimiter {
  constructor(cache) {
    this.cache = cache;
  }
  
  async checkRateLimit(identifier, maxRequests = 100, windowSeconds = 60) {
    const key = `ratelimit:${identifier}`;
    const current = await this.cache.increment(key);
    
    if (current === 1) {
      // First request in window, set expiration
      await this.cache.expire(key, windowSeconds);
    }
    
    if (current > maxRequests) {
      const ttl = await this.cache.ttl(key);
      throw new Error(`Rate limit exceeded. Try again in ${ttl} seconds`);
    }
    
    return {
      allowed: true,
      remaining: maxRequests - current,
      resetIn: await this.cache.ttl(key)
    };
  }
}

// 5. Distributed Locks
class DistributedLock {
  constructor(cache) {
    this.cache = cache;
  }
  
  async acquireLock(resource, ttl = 30) {
    const lockKey = `lock:${resource}`;
    const lockValue = this.generateLockId();
    
    // Try to acquire lock
    const acquired = await this.cache.client.set(lockKey, lockValue, {
      NX: true,  // Only set if not exists
      EX: ttl    // Expiration
    });
    
    if (acquired) {
      return lockValue;
    }
    
    return null;
  }
  
  async releaseLock(resource, lockValue) {
    const lockKey = `lock:${resource}`;
    
    // Only release if we own the lock
    const currentValue = await this.cache.get(lockKey);
    
    if (currentValue === lockValue) {
      await this.cache.delete(lockKey);
      return true;
    }
    
    return false;
  }
  
  async withLock(resource, callback, ttl = 30) {
    const lockValue = await this.acquireLock(resource, ttl);
    
    if (!lockValue) {
      throw new Error('Failed to acquire lock');
    }
    
    try {
      return await callback();
    } finally {
      await this.releaseLock(resource, lockValue);
    }
  }
  
  generateLockId() {
    return `lock_${Date.now()}_${Math.random().toString(36).substr(2, 16)}`;
  }
}

// 6. Leaderboard with Sorted Sets
class Leaderboard {
  constructor(cache) {
    this.cache = cache;
    this.leaderboardKey = 'game:leaderboard';
  }
  
  async addScore(userId, score) {
    await this.cache.client.zAdd(this.leaderboardKey, {
      score: score,
      value: userId
    });
  }
  
  async getTopPlayers(count = 10) {
    const players = await this.cache.client.zRangeWithScores(
      this.leaderboardKey,
      0,
      count - 1,
      { REV: true }
    );
    
    return players.map(p => ({
      userId: p.value,
      score: p.score
    }));
  }
  
  async getUserRank(userId) {
    const rank = await this.cache.client.zRevRank(this.leaderboardKey, userId);
    return rank !== null ? rank + 1 : null;
  }
  
  async getUserScore(userId) {
    return await this.cache.client.zScore(this.leaderboardKey, userId);
  }
}

// Usage Examples
const userRepo = new UserRepository(database, cache);
const user = await userRepo.getUser('123');

const sessionMgr = new SessionManager(cache);
const sessionId = await sessionMgr.createSession('user123', { theme: 'dark' });

const rateLimiter = new RateLimiter(cache);
await rateLimiter.checkRateLimit('user:123', 100, 60);

const lock = new DistributedLock(cache);
await lock.withLock('critical-resource', async () => {
  // Critical section
  console.log('Executing critical operation');
});

const leaderboard = new Leaderboard(cache);
await leaderboard.addScore('user123', 9500);
const topPlayers = await leaderboard.getTopPlayers(10);
```

### Best Practices:

1. ✅ Use appropriate cache eviction policies
2. ✅ Implement cache warming for critical data
3. ✅ Set reasonable TTLs based on data freshness needs
4. ✅ Handle cache failures gracefully
5. ✅ Use connection pooling
6. ✅ Monitor cache hit ratios
7. ✅ Implement cache stampede protection
8. ✅ Use Redis Cluster for large datasets
9. ✅ Enable automatic backups
10. ✅ Use Redis for more than just caching

---

## Question 12: Explain AWS CloudFormation for Infrastructure as Code

### 📋 Answer

**AWS CloudFormation** lets you model and provision AWS resources using infrastructure as code (IaC).

### Key Concepts:

1. **Templates**: JSON/YAML files defining resources
2. **Stacks**: Collection of AWS resources
3. **Change Sets**: Preview changes before applying
4. **Nested Stacks**: Modular template reuse
5. **Stack Sets**: Deploy across multiple accounts/regions

### Complete CloudFormation Template:

```yaml
# complete-app-stack.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Complete application stack with VPC, EC2, RDS, and S3'

Parameters:
  EnvironmentName:
    Type: String
    Default: Production
    AllowedValues:
      - Development
      - Staging
      - Production
    Description: Environment name
  
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
    Description: EC2 instance type
  
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
    Description: Database password

Resources:
  # VPC
  AppVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-VPC'
  
  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-IGW'
  
  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref AppVPC
      InternetGatewayId: !Ref InternetGateway
  
  # Public Subnet
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref AppVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Public-Subnet'
  
  # Private Subnet
  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref AppVPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Private-Subnet'
  
  # Route Table
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref AppVPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Public-RT'
  
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  
  SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable
  
  # Security Groups
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for web servers
      VpcId: !Ref AppVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-WebServer-SG'
  
  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for database
      VpcId: !Ref AppVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref WebServerSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Database-SG'
  
  # S3 Bucket
  AppDataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::StackName}-data-${AWS::AccountId}'
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
  
  # IAM Role for EC2
  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
  
  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref EC2Role
  
  # EC2 Instance
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-0c55b159cbfafe1f0
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      IamInstanceProfile: !Ref EC2InstanceProfile
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Hello from ${EnvironmentName}</h1>" > /var/www/html/index.html
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-WebServer'
  
  # RDS Subnet Group
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Subnet group for RDS
      SubnetIds:
        - !Ref PrivateSubnet
        - !Ref PublicSubnet
  
  # RDS Database
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: !Sub '${EnvironmentName}-database'
      Engine: postgres
      EngineVersion: '15.3'
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      StorageType: gp3
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref DatabaseSecurityGroup
      BackupRetentionPeriod: 7
      PreferredBackupWindow: '03:00-04:00'
      PreferredMaintenanceWindow: 'sun:04:00-sun:05:00'
      StorageEncrypted: true
      DeletionProtection: true
  
  # CloudWatch Alarm
  HighCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Alert when CPU exceeds 80%
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: InstanceId
          Value: !Ref WebServer

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref AppVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPC-ID'
  
  WebServerPublicIP:
    Description: Web server public IP
    Value: !GetAtt WebServer.PublicIp
  
  DatabaseEndpoint:
    Description: Database endpoint
    Value: !GetAtt Database.Endpoint.Address
  
  S3BucketName:
    Description: S3 bucket name
    Value: !Ref AppDataBucket
```

### Manage Stacks with AWS SDK:

```javascript
// cloudformation-manager.js
import {
  CloudFormationClient,
  CreateStackCommand,
  DescribeStacksCommand,
  UpdateStackCommand,
  DeleteStackCommand,
  ListStacksCommand,
  DescribeStackEventsCommand,
  ValidateTemplateCommand
} from '@aws-sdk/client-cloudformation';
import { readFileSync } from 'fs';

const cfClient = new CloudFormationClient({ region: 'us-east-1' });

// Create Stack
async function createStack(stackName, templatePath, parameters) {
  const template = readFileSync(templatePath, 'utf8');
  
  const command = new CreateStackCommand({
    StackName: stackName,
    TemplateBody: template,
    Parameters: Object.entries(parameters).map(([key, value]) => ({
      ParameterKey: key,
      ParameterValue: value
    })),
    Capabilities: ['CAPABILITY_IAM'],
    Tags: [
      { Key: 'Environment', Value: 'Production' },
      { Key: 'ManagedBy', Value: 'CloudFormation' }
    ]
  });
  
  try {
    const response = await cfClient.send(command);
    console.log('Stack creation initiated:', response.StackId);
    
    // Wait for completion
    await waitForStack(stackName, 'CREATE_COMPLETE');
    
    return response.StackId;
  } catch (error) {
    console.error('Failed to create stack:', error);
    throw error;
  }
}

// Wait for stack operation to complete
async function waitForStack(stackName, targetStatus) {
  const maxAttempts = 120;
  const delaySeconds = 10;
  
  for (let i = 0; i < maxAttempts; i++) {
    const stack = await getStackStatus(stackName);
    
    console.log(`Stack status: ${stack.StackStatus}`);
    
    if (stack.StackStatus === targetStatus) {
      console.log('Stack operation completed successfully');
      return stack;
    }
    
    if (stack.StackStatus.includes('FAILED') || stack.StackStatus.includes('ROLLBACK')) {
      throw new Error(`Stack operation failed: ${stack.StackStatus}`);
    }
    
    await new Promise(resolve => setTimeout(resolve, delaySeconds * 1000));
  }
  
  throw new Error('Stack operation timed out');
}

// Get stack status
async function getStackStatus(stackName) {
  const command = new DescribeStacksCommand({ StackName: stackName });
  const response = await cfClient.send(command);
  return response.Stacks[0];
}

// Usage
await createStack('MyAppStack', './complete-app-stack.yaml', {
  EnvironmentName: 'Production',
  InstanceType: 't3.small',
  DBPassword: 'SecurePassword123!'
});
```

### Best Practices:

1. ✅ Use parameters for flexibility
2. ✅ Organize with nested stacks
3. ✅ Version control templates
4. ✅ Use change sets for updates
5. ✅ Tag all resources
6. ✅ Enable termination protection
7. ✅ Use stack policies
8. ✅ Validate templates before deployment
9. ✅ Use cross-stack references
10. ✅ Implement rollback triggers

---

## Key Takeaways

### Question 10 (DynamoDB):
- NoSQL database with single-digit millisecond latency
- Partition key + optional sort key for flexible queries
- Global Secondary Indexes for additional access patterns
- DynamoDB Streams for change data capture
- Transactions for ACID operations

### Question 11 (ElastiCache):
- In-memory caching with Redis/Memcached
- Redis offers persistence, replication, complex data types
- Common patterns: cache-aside, write-through, session store
- Rate limiting and distributed locks
- Monitor cache hit ratios for effectiveness

### Question 12 (CloudFormation):
- Infrastructure as Code for AWS resources
- YAML/JSON templates define entire stack
- Parameters for reusable templates
- Change sets to preview modifications
- Outputs for cross-stack references
