# APIs Interview Questions (Q36-Q38): Analytics & Reporting

## Q36: How do you implement API analytics and usage tracking for business intelligence?

**Answer:**

```javascript
// analytics-service.js
const { InfluxDB, Point } = require('@influxdata/influxdb-client');
const { ClickHouse } = require('clickhouse');

class APIAnalyticsService {
    constructor() {
        this.influx = new InfluxDB({
            url: process.env.INFLUX_URL,
            token: process.env.INFLUX_TOKEN
        });
        this.writeApi = this.influx.getWriteApi('enbd', 'api_metrics');
        
        this.clickhouse = new ClickHouse({
            url: process.env.CLICKHOUSE_URL,
            database: 'analytics'
        });
    }

    async trackAPIRequest(req, res, duration) {
        const point = new Point('api_request')
            .tag('method', req.method)
            .tag('endpoint', req.route?.path || req.path)
            .tag('status_code', res.statusCode.toString())
            .tag('client_id', req.client?.id || 'anonymous')
            .intField('duration_ms', duration)
            .intField('request_size', req.headers['content-length'] || 0)
            .intField('response_size', res.get('content-length') || 0)
            .timestamp(new Date());

        this.writeApi.writePoint(point);
        await this.writeApi.flush();

        await this.clickhouse.insert('INSERT INTO api_requests', {
            timestamp: new Date(),
            method: req.method,
            endpoint: req.path,
            status_code: res.statusCode,
            client_id: req.client?.id || 'anonymous',
            duration_ms: duration,
            user_agent: req.get('user-agent'),
            ip_address: req.ip
        }).toPromise();
    }

    async getUsageStats(clientId, timeRange = '24h') {
        const query = `
            from(bucket: "api_metrics")
            |> range(start: -${timeRange})
            |> filter(fn: (r) => r._measurement == "api_request")
            |> filter(fn: (r) => r.client_id == "${clientId}")
            |> group(columns: ["endpoint"])
            |> count()
        `;

        const result = await this.influx.getQueryApi('enbd').collectRows(query);
        return result;
    }

    async getTopEndpoints(limit = 10) {
        const query = `
            SELECT 
                endpoint,
                COUNT(*) as request_count,
                AVG(duration_ms) as avg_duration,
                percentile(duration_ms, 0.95) as p95_duration,
                percentile(duration_ms, 0.99) as p99_duration
            FROM api_requests
            WHERE timestamp > NOW() - INTERVAL 1 DAY
            GROUP BY endpoint
            ORDER BY request_count DESC
            LIMIT ${limit}
        `;

        return this.clickhouse.query(query).toPromise();
    }

    async getErrorRate(timeWindow = '1h') {
        const query = `
            SELECT 
                toStartOfHour(timestamp) as hour,
                countIf(status_code >= 400) / count() * 100 as error_rate
            FROM api_requests
            WHERE timestamp > NOW() - INTERVAL ${timeWindow}
            GROUP BY hour
            ORDER BY hour
        `;

        return this.clickhouse.query(query).toPromise();
    }
}

module.exports = { APIAnalyticsService };
```

---

## Q37: How do you create real-time dashboards for API monitoring and business metrics?

**Answer:**

```javascript
// dashboard-service.js
const WebSocket = require('ws');
const Redis = require('ioredis');

class DashboardService {
    constructor() {
        this.redis = new Redis();
        this.wss = new WebSocket.Server({ port: 8080 });
        this.setupWebSocket();
    }

    setupWebSocket() {
        this.wss.on('connection', (ws, req) => {
            console.log('Dashboard connected');

            ws.on('message', async (message) => {
                const data = JSON.parse(message);
                
                if (data.type === 'subscribe') {
                    await this.subscribeToMetrics(ws, data.metrics);
                }
            });

            ws.on('close', () => {
                console.log('Dashboard disconnected');
            });
        });
    }

    async subscribeToMetrics(ws, metrics) {
        const subscriber = new Redis();
        
        for (const metric of metrics) {
            subscriber.subscribe(`metrics:${metric}`);
        }

        subscriber.on('message', (channel, message) => {
            ws.send(JSON.stringify({
                channel,
                data: JSON.parse(message),
                timestamp: new Date().toISOString()
            }));
        });
    }

    async publishMetric(metric, data) {
        await this.redis.publish(
            `metrics:${metric}`,
            JSON.stringify(data)
        );
    }

    async getRealTimeMetrics() {
        return {
            requests_per_second: await this.getRequestsPerSecond(),
            active_users: await this.getActiveUsers(),
            avg_response_time: await this.getAvgResponseTime(),
            error_rate: await this.getErrorRate(),
            database_connections: await this.getDatabaseConnections()
        };
    }

    async getRequestsPerSecond() {
        const count = await this.redis.get('metrics:requests:1s');
        return parseInt(count || 0);
    }

    async getActiveUsers() {
        return this.redis.scard('active_users');
    }

    async getAvgResponseTime() {
        const times = await this.redis.lrange('response_times', 0, 99);
        const avg = times.reduce((sum, t) => sum + parseInt(t), 0) / times.length;
        return Math.round(avg);
    }

    async getErrorRate() {
        const total = await this.redis.get('metrics:requests:total') || 0;
        const errors = await this.redis.get('metrics:errors:total') || 0;
        return total > 0 ? (errors / total * 100).toFixed(2) : 0;
    }

    async getDatabaseConnections() {
        const result = await db.query('SELECT count(*) FROM pg_stat_activity');
        return parseInt(result.rows[0].count);
    }
}

module.exports = { DashboardService };
```

---

## Q38: How do you implement API usage billing and quota management?

**Answer:**

```javascript
// billing-service.js
class BillingService {
    constructor(db, redis) {
        this.db = db;
        this.redis = redis;
        
        this.pricingTiers = {
            free: { requests: 1000, price: 0 },
            basic: { requests: 100000, price: 49 },
            premium: { requests: 1000000, price: 199 },
            enterprise: { requests: -1, price: 999 }
        };
    }

    async trackAPIUsage(clientId, endpoint, cost = 1) {
        const key = `usage:${clientId}:${this.getCurrentMonth()}`;
        const usage = await this.redis.hincrby(key, endpoint, cost);
        
        await this.redis.expire(key, 60 * 60 * 24 * 60); // 60 days
        
        const quota = await this.getClientQuota(clientId);
        if (quota > 0 && usage > quota) {
            throw new Error('Quota exceeded');
        }
        
        return { usage, quota, remaining: quota - usage };
    }

    async getClientQuota(clientId) {
        const result = await this.db.query(
            'SELECT tier FROM clients WHERE id = $1',
            [clientId]
        );
        
        const tier = result.rows[0]?.tier || 'free';
        return this.pricingTiers[tier].requests;
    }

    async getMonthlyUsage(clientId) {
        const key = `usage:${clientId}:${this.getCurrentMonth()}`;
        const usage = await this.redis.hgetall(key);
        
        const total = Object.values(usage).reduce(
            (sum, count) => sum + parseInt(count), 0
        );
        
        return { total, byEndpoint: usage };
    }

    async generateInvoice(clientId, month) {
        const usage = await this.getMonthlyUsage(clientId);
        const client = await this.getClient(clientId);
        const tier = this.pricingTiers[client.tier];
        
        let overageCharge = 0;
        if (tier.requests > 0 && usage.total > tier.requests) {
            const overage = usage.total - tier.requests;
            overageCharge = overage * 0.001; // $0.001 per request
        }
        
        const invoice = {
            clientId,
            month,
            tier: client.tier,
            baseCharge: tier.price,
            requests: usage.total,
            overageRequests: Math.max(0, usage.total - tier.requests),
            overageCharge,
            totalAmount: tier.price + overageCharge,
            currency: 'USD',
            createdAt: new Date()
        };
        
        await this.db.query(`
            INSERT INTO invoices (
                client_id, month, tier, base_charge, requests,
                overage_charge, total_amount, created_at
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
        `, [
            invoice.clientId,
            invoice.month,
            invoice.tier,
            invoice.baseCharge,
            invoice.requests,
            invoice.overageCharge,
            invoice.totalAmount
        ]);
        
        return invoice;
    }

    getCurrentMonth() {
        const now = new Date();
        return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;
    }

    async getClient(clientId) {
        const result = await this.db.query(
            'SELECT * FROM clients WHERE id = $1',
            [clientId]
        );
        return result.rows[0];
    }
}

module.exports = { BillingService };
```

---
