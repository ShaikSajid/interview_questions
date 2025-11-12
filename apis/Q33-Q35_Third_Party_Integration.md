# APIs Interview Questions (Q33-Q35): Third-Party Integration

## Q33: How do you integrate payment gateways (Stripe, PayPal) in banking APIs with proper error handling and reconciliation?

**Answer:**

Payment gateway integration requires robust error handling, retry logic, idempotency, and transaction reconciliation for financial accuracy.

### Payment Gateway Integration:

```javascript
// payment-gateway-service.js
const Stripe = require('stripe');
const axios = require('axios');
const { v4: uuidv4 } = require('uuid');

/**
 * Payment Gateway Manager
 */
class PaymentGatewayManager {
    constructor(config) {
        this.stripe = new Stripe(config.stripeSecretKey);
        this.paypalClientId = config.paypalClientId;
        this.paypalSecret = config.paypalSecret;
        this.paypalBaseURL = config.paypalSandbox 
            ? 'https://api-m.sandbox.paypal.com'
            : 'https://api-m.paypal.com';
    }

    /**
     * Create payment intent (Stripe)
     */
    async createStripePayment(paymentData) {
        const {
            amount,
            currency = 'AED',
            customerId,
            description,
            metadata = {}
        } = paymentData;

        // Generate idempotency key to prevent duplicate charges
        const idempotencyKey = uuidv4();

        try {
            const paymentIntent = await this.stripe.paymentIntents.create({
                amount: Math.round(amount * 100), // Convert to cents
                currency: currency.toLowerCase(),
                customer: customerId,
                description,
                metadata: {
                    ...metadata,
                    bank: 'Emirates NBD',
                    timestamp: new Date().toISOString()
                },
                automatic_payment_methods: {
                    enabled: true
                }
            }, {
                idempotencyKey
            });

            // Log payment intent creation
            await this.logPaymentEvent({
                gateway: 'stripe',
                type: 'payment_intent_created',
                intentId: paymentIntent.id,
                amount,
                currency,
                status: paymentIntent.status
            });

            return {
                success: true,
                paymentId: paymentIntent.id,
                clientSecret: paymentIntent.client_secret,
                status: paymentIntent.status,
                amount: paymentIntent.amount / 100
            };

        } catch (error) {
            await this.handlePaymentError('stripe', error, paymentData);
            throw error;
        }
    }

    /**
     * Confirm payment (Stripe)
     */
    async confirmStripePayment(paymentIntentId, paymentMethodId) {
        try {
            const paymentIntent = await this.stripe.paymentIntents.confirm(
                paymentIntentId,
                { payment_method: paymentMethodId }
            );

            await this.logPaymentEvent({
                gateway: 'stripe',
                type: 'payment_confirmed',
                intentId: paymentIntent.id,
                status: paymentIntent.status
            });

            return {
                success: true,
                paymentId: paymentIntent.id,
                status: paymentIntent.status,
                amount: paymentIntent.amount / 100
            };

        } catch (error) {
            await this.handlePaymentError('stripe', error, { paymentIntentId });
            throw error;
        }
    }

    /**
     * Refund payment (Stripe)
     */
    async refundStripePayment(paymentIntentId, amount = null, reason = null) {
        try {
            const refundData = {
                payment_intent: paymentIntentId
            };

            if (amount) {
                refundData.amount = Math.round(amount * 100);
            }

            if (reason) {
                refundData.reason = reason; // 'duplicate', 'fraudulent', 'requested_by_customer'
            }

            const refund = await this.stripe.refunds.create(refundData);

            await this.logPaymentEvent({
                gateway: 'stripe',
                type: 'refund_created',
                refundId: refund.id,
                paymentIntentId,
                amount: refund.amount / 100,
                status: refund.status
            });

            return {
                success: true,
                refundId: refund.id,
                status: refund.status,
                amount: refund.amount / 100
            };

        } catch (error) {
            await this.handlePaymentError('stripe', error, { paymentIntentId, amount });
            throw error;
        }
    }

    /**
     * Create PayPal order
     */
    async createPayPalPayment(paymentData) {
        const {
            amount,
            currency = 'USD',
            description,
            returnUrl,
            cancelUrl
        } = paymentData;

        try {
            const accessToken = await this.getPayPalAccessToken();

            const orderData = {
                intent: 'CAPTURE',
                purchase_units: [{
                    amount: {
                        currency_code: currency,
                        value: amount.toFixed(2)
                    },
                    description
                }],
                application_context: {
                    return_url: returnUrl,
                    cancel_url: cancelUrl,
                    brand_name: 'Emirates NBD',
                    shipping_preference: 'NO_SHIPPING',
                    user_action: 'PAY_NOW'
                }
            };

            const response = await axios.post(
                `${this.paypalBaseURL}/v2/checkout/orders`,
                orderData,
                {
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${accessToken}`
                    }
                }
            );

            await this.logPaymentEvent({
                gateway: 'paypal',
                type: 'order_created',
                orderId: response.data.id,
                amount,
                currency,
                status: response.data.status
            });

            // Get approval URL
            const approvalUrl = response.data.links.find(
                link => link.rel === 'approve'
            )?.href;

            return {
                success: true,
                orderId: response.data.id,
                approvalUrl,
                status: response.data.status
            };

        } catch (error) {
            await this.handlePaymentError('paypal', error, paymentData);
            throw error;
        }
    }

    /**
     * Capture PayPal order
     */
    async capturePayPalPayment(orderId) {
        try {
            const accessToken = await this.getPayPalAccessToken();

            const response = await axios.post(
                `${this.paypalBaseURL}/v2/checkout/orders/${orderId}/capture`,
                {},
                {
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `Bearer ${accessToken}`
                    }
                }
            );

            const capture = response.data.purchase_units[0].payments.captures[0];

            await this.logPaymentEvent({
                gateway: 'paypal',
                type: 'payment_captured',
                orderId,
                captureId: capture.id,
                amount: capture.amount.value,
                status: capture.status
            });

            return {
                success: true,
                orderId,
                captureId: capture.id,
                status: capture.status,
                amount: parseFloat(capture.amount.value)
            };

        } catch (error) {
            await this.handlePaymentError('paypal', error, { orderId });
            throw error;
        }
    }

    /**
     * Get PayPal access token
     */
    async getPayPalAccessToken() {
        const auth = Buffer.from(
            `${this.paypalClientId}:${this.paypalSecret}`
        ).toString('base64');

        const response = await axios.post(
            `${this.paypalBaseURL}/v1/oauth2/token`,
            'grant_type=client_credentials',
            {
                headers: {
                    'Authorization': `Basic ${auth}`,
                    'Content-Type': 'application/x-www-form-urlencoded'
                }
            }
        );

        return response.data.access_token;
    }

    /**
     * Handle payment errors with retry logic
     */
    async handlePaymentError(gateway, error, paymentData) {
        const errorDetails = {
            gateway,
            error: error.message,
            code: error.code || error.response?.status,
            data: paymentData,
            timestamp: new Date().toISOString()
        };

        // Log error
        console.error(`Payment error (${gateway}):`, errorDetails);

        // Store in database for investigation
        await db.query(`
            INSERT INTO payment_errors (
                gateway, error_type, error_message, error_code, payment_data, created_at
            ) VALUES ($1, $2, $3, $4, $5, NOW())
        `, [
            gateway,
            error.type || 'unknown',
            error.message,
            error.code || error.response?.status,
            JSON.stringify(paymentData)
        ]);

        // Determine if error is retryable
        const retryableErrors = [
            'rate_limit',
            'api_connection_error',
            'network_error',
            503, // Service unavailable
            504  // Gateway timeout
        ];

        const isRetryable = retryableErrors.some(
            code => error.code === code || error.response?.status === code
        );

        if (isRetryable) {
            errorDetails.retryable = true;
            errorDetails.retryAfter = error.response?.headers['retry-after'] || 60;
        }

        return errorDetails;
    }

    /**
     * Log payment events for audit trail
     */
    async logPaymentEvent(event) {
        await db.query(`
            INSERT INTO payment_events (
                gateway, event_type, payment_id, amount, currency, status, metadata, created_at
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
        `, [
            event.gateway,
            event.type,
            event.intentId || event.orderId || event.refundId || event.captureId,
            event.amount,
            event.currency,
            event.status,
            JSON.stringify(event)
        ]);
    }
}

/**
 * Payment Reconciliation Service
 */
class PaymentReconciliationService {
    constructor(paymentGateway) {
        this.gateway = paymentGateway;
    }

    /**
     * Reconcile payments with gateway
     */
    async reconcilePayments(date) {
        console.log(`🔍 Reconciling payments for ${date}`);

        // Get payments from our database
        const ourPayments = await this.getOurPayments(date);
        
        // Get payments from Stripe
        const stripePayments = await this.getStripePayments(date);
        
        // Compare and find discrepancies
        const discrepancies = this.comparePayments(ourPayments, stripePayments);

        if (discrepancies.length > 0) {
            console.log(`⚠️ Found ${discrepancies.length} discrepancies`);
            await this.handleDiscrepancies(discrepancies);
        } else {
            console.log('✅ All payments reconciled successfully');
        }

        return {
            date,
            totalPayments: ourPayments.length,
            discrepancies: discrepancies.length,
            reconciled: ourPayments.length - discrepancies.length
        };
    }

    /**
     * Get payments from our database
     */
    async getOurPayments(date) {
        const result = await db.query(`
            SELECT 
                payment_id,
                gateway,
                amount,
                currency,
                status,
                created_at
            FROM payments
            WHERE DATE(created_at) = $1
            ORDER BY created_at
        `, [date]);

        return result.rows;
    }

    /**
     * Get payments from Stripe
     */
    async getStripePayments(date) {
        const startDate = new Date(date);
        startDate.setHours(0, 0, 0, 0);
        
        const endDate = new Date(date);
        endDate.setHours(23, 59, 59, 999);

        const paymentIntents = await this.gateway.stripe.paymentIntents.list({
            created: {
                gte: Math.floor(startDate.getTime() / 1000),
                lte: Math.floor(endDate.getTime() / 1000)
            },
            limit: 100
        });

        return paymentIntents.data.map(intent => ({
            payment_id: intent.id,
            gateway: 'stripe',
            amount: intent.amount / 100,
            currency: intent.currency.toUpperCase(),
            status: intent.status,
            created_at: new Date(intent.created * 1000)
        }));
    }

    /**
     * Compare payments and find discrepancies
     */
    comparePayments(ourPayments, gatewayPayments) {
        const discrepancies = [];

        const gatewayMap = new Map(
            gatewayPayments.map(p => [p.payment_id, p])
        );

        for (const ourPayment of ourPayments) {
            const gatewayPayment = gatewayMap.get(ourPayment.payment_id);

            if (!gatewayPayment) {
                discrepancies.push({
                    type: 'missing_in_gateway',
                    payment: ourPayment
                });
            } else if (ourPayment.amount !== gatewayPayment.amount) {
                discrepancies.push({
                    type: 'amount_mismatch',
                    payment: ourPayment,
                    gatewayPayment
                });
            } else if (ourPayment.status !== gatewayPayment.status) {
                discrepancies.push({
                    type: 'status_mismatch',
                    payment: ourPayment,
                    gatewayPayment
                });
            }
        }

        return discrepancies;
    }

    /**
     * Handle payment discrepancies
     */
    async handleDiscrepancies(discrepancies) {
        for (const discrepancy of discrepancies) {
            console.log(`⚠️ Discrepancy: ${discrepancy.type}`, discrepancy.payment);

            // Store discrepancy for investigation
            await db.query(`
                INSERT INTO payment_discrepancies (
                    payment_id, discrepancy_type, our_data, gateway_data, created_at
                ) VALUES ($1, $2, $3, $4, NOW())
            `, [
                discrepancy.payment.payment_id,
                discrepancy.type,
                JSON.stringify(discrepancy.payment),
                JSON.stringify(discrepancy.gatewayPayment)
            ]);

            // Send alert to finance team
            await this.sendDiscrepancyAlert(discrepancy);
        }
    }

    async sendDiscrepancyAlert(discrepancy) {
        // Send email/notification to finance team
        console.log(`📧 Alert sent for discrepancy: ${discrepancy.type}`);
    }

    /**
     * Schedule daily reconciliation
     */
    scheduleDailyReconciliation() {
        const cron = require('node-cron');

        // Run at 2 AM daily
        cron.schedule('0 2 * * *', async () => {
            const yesterday = new Date();
            yesterday.setDate(yesterday.getDate() - 1);
            const date = yesterday.toISOString().split('T')[0];

            try {
                await this.reconcilePayments(date);
            } catch (error) {
                console.error('Reconciliation failed:', error);
            }
        });
    }
}

/**
 * Webhook Handler for Payment Events
 */
class PaymentWebhookHandler {
    constructor(paymentGateway) {
        this.gateway = paymentGateway;
    }

    /**
     * Handle Stripe webhook
     */
    async handleStripeWebhook(req, res) {
        const sig = req.headers['stripe-signature'];
        const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;

        let event;

        try {
            event = this.gateway.stripe.webhooks.constructEvent(
                req.body,
                sig,
                webhookSecret
            );
        } catch (error) {
            console.error('Webhook signature verification failed:', error.message);
            return res.status(400).send(`Webhook Error: ${error.message}`);
        }

        // Handle event
        switch (event.type) {
            case 'payment_intent.succeeded':
                await this.handlePaymentSuccess(event.data.object);
                break;

            case 'payment_intent.payment_failed':
                await this.handlePaymentFailure(event.data.object);
                break;

            case 'charge.refunded':
                await this.handleRefund(event.data.object);
                break;

            case 'payment_intent.canceled':
                await this.handlePaymentCanceled(event.data.object);
                break;

            default:
                console.log(`Unhandled event type: ${event.type}`);
        }

        res.json({ received: true });
    }

    async handlePaymentSuccess(paymentIntent) {
        console.log(`✅ Payment succeeded: ${paymentIntent.id}`);

        // Update payment status in database
        await db.query(`
            UPDATE payments
            SET status = 'succeeded',
                updated_at = NOW()
            WHERE payment_id = $1
        `, [paymentIntent.id]);

        // Update account balance
        await db.query(`
            UPDATE accounts
            SET balance = balance + $1,
                updated_at = NOW()
            WHERE id = $2
        `, [paymentIntent.amount / 100, paymentIntent.metadata.accountId]);

        // Send confirmation email
        await this.sendPaymentConfirmation(paymentIntent);
    }

    async handlePaymentFailure(paymentIntent) {
        console.log(`❌ Payment failed: ${paymentIntent.id}`);

        // Update payment status
        await db.query(`
            UPDATE payments
            SET status = 'failed',
                error_message = $2,
                updated_at = NOW()
            WHERE payment_id = $1
        `, [paymentIntent.id, paymentIntent.last_payment_error?.message]);

        // Notify customer
        await this.sendPaymentFailureNotification(paymentIntent);
    }

    async handleRefund(charge) {
        console.log(`🔄 Refund processed: ${charge.id}`);

        // Record refund
        await db.query(`
            INSERT INTO refunds (
                charge_id, amount, currency, reason, created_at
            ) VALUES ($1, $2, $3, $4, NOW())
        `, [charge.id, charge.amount_refunded / 100, charge.currency, charge.refund?.reason]);

        // Update account balance
        await db.query(`
            UPDATE accounts
            SET balance = balance - $1,
                updated_at = NOW()
            WHERE id = $2
        `, [charge.amount_refunded / 100, charge.metadata.accountId]);
    }

    async handlePaymentCanceled(paymentIntent) {
        console.log(`🚫 Payment canceled: ${paymentIntent.id}`);

        await db.query(`
            UPDATE payments
            SET status = 'canceled',
                updated_at = NOW()
            WHERE payment_id = $1
        `, [paymentIntent.id]);
    }

    async sendPaymentConfirmation(paymentIntent) {
        // Send email confirmation
        console.log(`📧 Payment confirmation sent for ${paymentIntent.id}`);
    }

    async sendPaymentFailureNotification(paymentIntent) {
        // Send failure notification
        console.log(`📧 Payment failure notification sent for ${paymentIntent.id}`);
    }
}

// Usage Example
const paymentGateway = new PaymentGatewayManager({
    stripeSecretKey: process.env.STRIPE_SECRET_KEY,
    paypalClientId: process.env.PAYPAL_CLIENT_ID,
    paypalSecret: process.env.PAYPAL_SECRET,
    paypalSandbox: process.env.NODE_ENV !== 'production'
});

const reconciliationService = new PaymentReconciliationService(paymentGateway);
const webhookHandler = new PaymentWebhookHandler(paymentGateway);

// Express routes
const express = require('express');
const app = express();

// Create payment
app.post('/api/payments', async (req, res) => {
    try {
        const { amount, currency, customerId, description } = req.body;

        const payment = await paymentGateway.createStripePayment({
            amount,
            currency,
            customerId,
            description
        });

        res.json(payment);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Webhook endpoint
app.post('/webhooks/stripe',
    express.raw({ type: 'application/json' }),
    (req, res) => webhookHandler.handleStripeWebhook(req, res)
);

// Schedule reconciliation
reconciliationService.scheduleDailyReconciliation();

module.exports = {
    PaymentGatewayManager,
    PaymentReconciliationService,
    PaymentWebhookHandler
};
```

---

## Q34: How do you design and implement SDKs for banking APIs with multiple language support?

**Answer:**

SDKs simplify API integration for developers by providing language-specific wrappers, type safety, and better developer experience.

### SDK Design & Implementation:

```javascript
// sdk/nodejs/enbd-banking-sdk.js

/**
 * Emirates NBD Banking SDK for Node.js
 */
class ENBDBankingSDK {
    constructor(config) {
        this.apiKey = config.apiKey;
        this.baseURL = config.baseURL || 'https://api.emiratesnbd.com/v1';
        this.timeout = config.timeout || 30000;
        this.maxRetries = config.maxRetries || 3;
        
        this.axios = require('axios').create({
            baseURL: this.baseURL,
            timeout: this.timeout,
            headers: {
                'X-API-Key': this.apiKey,
                'Content-Type': 'application/json',
                'User-Agent': `ENBD-SDK-Node/${this.getVersion()}`
            }
        });

        // Add interceptors
        this.setupInterceptors();
    }

    /**
     * Setup axios interceptors
     */
    setupInterceptors() {
        // Request interceptor
        this.axios.interceptors.request.use(
            (config) => {
                config.headers['X-Request-ID'] = this.generateRequestId();
                config.metadata = { startTime: Date.now() };
                return config;
            },
            (error) => Promise.reject(error)
        );

        // Response interceptor
        this.axios.interceptors.response.use(
            (response) => {
                const duration = Date.now() - response.config.metadata.startTime;
                console.log(`Request completed in ${duration}ms`);
                return response.data;
            },
            async (error) => {
                return this.handleError(error);
            }
        );
    }

    /**
     * Handle API errors with retry logic
     */
    async handleError(error) {
        const config = error.config;

        // Don't retry if already exceeded max retries
        if (!config || config.__retryCount >= this.maxRetries) {
            throw this.formatError(error);
        }

        config.__retryCount = config.__retryCount || 0;
        config.__retryCount++;

        // Retry on specific errors
        const retryableStatuses = [408, 429, 500, 502, 503, 504];
        if (retryableStatuses.includes(error.response?.status)) {
            const delay = this.getRetryDelay(config.__retryCount);
            console.log(`Retrying request (${config.__retryCount}/${this.maxRetries}) after ${delay}ms`);
            
            await this.sleep(delay);
            return this.axios(config);
        }

        throw this.formatError(error);
    }

    /**
     * Format error response
     */
    formatError(error) {
        if (error.response) {
            return new ENBDAPIError(
                error.response.data.message || 'API Error',
                error.response.status,
                error.response.data.error,
                error.response.data
            );
        } else if (error.request) {
            return new ENBDAPIError(
                'No response from server',
                0,
                'NetworkError'
            );
        } else {
            return new ENBDAPIError(
                error.message,
                0,
                'RequestError'
            );
        }
    }

    /**
     * Customers API
     */
    get customers() {
        return {
            list: async (params = {}) => {
                return this.axios.get('/customers', { params });
            },

            get: async (customerId) => {
                return this.axios.get(`/customers/${customerId}`);
            },

            create: async (customerData) => {
                return this.axios.post('/customers', customerData);
            },

            update: async (customerId, updates) => {
                return this.axios.patch(`/customers/${customerId}`, updates);
            },

            delete: async (customerId) => {
                return this.axios.delete(`/customers/${customerId}`);
            }
        };
    }

    /**
     * Accounts API
     */
    get accounts() {
        return {
            list: async (customerId, params = {}) => {
                return this.axios.get(`/customers/${customerId}/accounts`, { params });
            },

            get: async (accountId) => {
                return this.axios.get(`/accounts/${accountId}`);
            },

            create: async (customerId, accountData) => {
                return this.axios.post(`/customers/${customerId}/accounts`, accountData);
            },

            getBalance: async (accountId) => {
                return this.axios.get(`/accounts/${accountId}/balance`);
            },

            freeze: async (accountId, reason) => {
                return this.axios.post(`/accounts/${accountId}/freeze`, { reason });
            },

            unfreeze: async (accountId) => {
                return this.axios.post(`/accounts/${accountId}/unfreeze`);
            }
        };
    }

    /**
     * Transactions API
     */
    get transactions() {
        return {
            list: async (accountId, params = {}) => {
                return this.axios.get(`/accounts/${accountId}/transactions`, { params });
            },

            get: async (transactionId) => {
                return this.axios.get(`/transactions/${transactionId}`);
            },

            create: async (accountId, transactionData) => {
                return this.axios.post(`/accounts/${accountId}/transactions`, transactionData);
            },

            transfer: async (fromAccountId, toAccountId, amount, description) => {
                return this.axios.post('/transactions/transfer', {
                    fromAccountId,
                    toAccountId,
                    amount,
                    description
                });
            }
        };
    }

    /**
     * Payments API
     */
    get payments() {
        return {
            create: async (paymentData) => {
                return this.axios.post('/payments', paymentData);
            },

            get: async (paymentId) => {
                return this.axios.get(`/payments/${paymentId}`);
            },

            cancel: async (paymentId) => {
                return this.axios.post(`/payments/${paymentId}/cancel`);
            },

            refund: async (paymentId, amount = null) => {
                return this.axios.post(`/payments/${paymentId}/refund`, { amount });
            }
        };
    }

    // Utility methods
    generateRequestId() {
        return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }

    getRetryDelay(retryCount) {
        return Math.min(1000 * Math.pow(2, retryCount), 10000); // Exponential backoff, max 10s
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    getVersion() {
        return '1.0.0';
    }
}

/**
 * Custom Error Class
 */
class ENBDAPIError extends Error {
    constructor(message, statusCode, errorCode, data = null) {
        super(message);
        this.name = 'ENBDAPIError';
        this.statusCode = statusCode;
        this.errorCode = errorCode;
        this.data = data;
    }
}

// Type definitions (TypeScript)
const typeDefinitions = `
// types/index.d.ts

export interface ENBDConfig {
    apiKey: string;
    baseURL?: string;
    timeout?: number;
    maxRetries?: number;
}

export interface Customer {
    id: string;
    name: string;
    email: string;
    phoneNumber: string;
    country: string;
    createdAt: Date;
}

export interface Account {
    id: string;
    customerId: string;
    accountNumber: string;
    accountType: 'savings' | 'current' | 'fixed_deposit';
    balance: number;
    currency: string;
    status: 'active' | 'frozen' | 'closed';
    createdAt: Date;
}

export interface Transaction {
    id: string;
    accountId: string;
    type: 'credit' | 'debit';
    amount: number;
    description: string;
    status: 'pending' | 'completed' | 'failed';
    createdAt: Date;
}

export interface Payment {
    id: string;
    amount: number;
    currency: string;
    status: 'pending' | 'succeeded' | 'failed' | 'canceled';
    createdAt: Date;
}

export declare class ENBDBankingSDK {
    constructor(config: ENBDConfig);
    
    customers: {
        list(params?: any): Promise<Customer[]>;
        get(customerId: string): Promise<Customer>;
        create(customerData: Partial<Customer>): Promise<Customer>;
        update(customerId: string, updates: Partial<Customer>): Promise<Customer>;
        delete(customerId: string): Promise<void>;
    };
    
    accounts: {
        list(customerId: string, params?: any): Promise<Account[]>;
        get(accountId: string): Promise<Account>;
        create(customerId: string, accountData: Partial<Account>): Promise<Account>;
        getBalance(accountId: string): Promise<{ balance: number }>;
        freeze(accountId: string, reason: string): Promise<Account>;
        unfreeze(accountId: string): Promise<Account>;
    };
    
    transactions: {
        list(accountId: string, params?: any): Promise<Transaction[]>;
        get(transactionId: string): Promise<Transaction>;
        create(accountId: string, transactionData: Partial<Transaction>): Promise<Transaction>;
        transfer(fromAccountId: string, toAccountId: string, amount: number, description: string): Promise<Transaction>;
    };
    
    payments: {
        create(paymentData: Partial<Payment>): Promise<Payment>;
        get(paymentId: string): Promise<Payment>;
        cancel(paymentId: string): Promise<Payment>;
        refund(paymentId: string, amount?: number): Promise<Payment>;
    };
}

export declare class ENBDAPIError extends Error {
    statusCode: number;
    errorCode: string;
    data: any;
}
`;

// Python SDK
const pythonSDK = `
# sdk/python/enbd_banking/__init__.py

import requests
import time
from typing import Dict, List, Optional
from datetime import datetime

class ENBDBankingSDK:
    """Emirates NBD Banking SDK for Python"""
    
    def __init__(self, api_key: str, base_url: str = "https://api.emiratesnbd.com/v1", 
                 timeout: int = 30, max_retries: int = 3):
        self.api_key = api_key
        self.base_url = base_url
        self.timeout = timeout
        self.max_retries = max_retries
        
        self.session = requests.Session()
        self.session.headers.update({
            'X-API-Key': api_key,
            'Content-Type': 'application/json',
            'User-Agent': f'ENBD-SDK-Python/{self._get_version()}'
        })
    
    def _request(self, method: str, endpoint: str, **kwargs) -> Dict:
        """Make HTTP request with retry logic"""
        url = f"{self.base_url}{endpoint}"
        
        for attempt in range(self.max_retries):
            try:
                response = self.session.request(
                    method, url, timeout=self.timeout, **kwargs
                )
                response.raise_for_status()
                return response.json()
                
            except requests.exceptions.HTTPError as e:
                if e.response.status_code in [408, 429, 500, 502, 503, 504]:
                    if attempt < self.max_retries - 1:
                        delay = min(2 ** attempt, 10)
                        time.sleep(delay)
                        continue
                raise ENBDAPIError.from_response(e.response)
                
            except requests.exceptions.RequestException as e:
                if attempt < self.max_retries - 1:
                    time.sleep(2 ** attempt)
                    continue
                raise ENBDAPIError(str(e), 0, 'NetworkError')
    
    # Customers API
    def list_customers(self, **params) -> List[Dict]:
        return self._request('GET', '/customers', params=params)
    
    def get_customer(self, customer_id: str) -> Dict:
        return self._request('GET', f'/customers/{customer_id}')
    
    def create_customer(self, customer_data: Dict) -> Dict:
        return self._request('POST', '/customers', json=customer_data)
    
    def update_customer(self, customer_id: str, updates: Dict) -> Dict:
        return self._request('PATCH', f'/customers/{customer_id}', json=updates)
    
    def delete_customer(self, customer_id: str) -> None:
        self._request('DELETE', f'/customers/{customer_id}')
    
    # Accounts API
    def list_accounts(self, customer_id: str, **params) -> List[Dict]:
        return self._request('GET', f'/customers/{customer_id}/accounts', params=params)
    
    def get_account(self, account_id: str) -> Dict:
        return self._request('GET', f'/accounts/{account_id}')
    
    def create_account(self, customer_id: str, account_data: Dict) -> Dict:
        return self._request('POST', f'/customers/{customer_id}/accounts', json=account_data)
    
    def get_account_balance(self, account_id: str) -> Dict:
        return self._request('GET', f'/accounts/{account_id}/balance')
    
    # Transactions API
    def list_transactions(self, account_id: str, **params) -> List[Dict]:
        return self._request('GET', f'/accounts/{account_id}/transactions', params=params)
    
    def create_transaction(self, account_id: str, transaction_data: Dict) -> Dict:
        return self._request('POST', f'/accounts/{account_id}/transactions', json=transaction_data)
    
    def transfer(self, from_account_id: str, to_account_id: str, 
                amount: float, description: str) -> Dict:
        return self._request('POST', '/transactions/transfer', json={
            'fromAccountId': from_account_id,
            'toAccountId': to_account_id,
            'amount': amount,
            'description': description
        })
    
    def _get_version(self) -> str:
        return '1.0.0'

class ENBDAPIError(Exception):
    """Custom exception for ENBD API errors"""
    
    def __init__(self, message: str, status_code: int, error_code: str, data: Dict = None):
        super().__init__(message)
        self.message = message
        self.status_code = status_code
        self.error_code = error_code
        self.data = data
    
    @classmethod
    def from_response(cls, response):
        data = response.json() if response.content else {}
        return cls(
            message=data.get('message', 'API Error'),
            status_code=response.status_code,
            error_code=data.get('error', 'UnknownError'),
            data=data
        )

# Usage example
if __name__ == '__main__':
    sdk = ENBDBankingSDK(api_key='enbd_your_api_key')
    
    # Get customers
    customers = sdk.list_customers(limit=10)
    print(f"Found {len(customers)} customers")
    
    # Get account balance
    balance = sdk.get_account_balance('acc_123456')
    print(f"Balance: {balance['balance']} {balance['currency']}")
`;

module.exports = {
    ENBDBankingSDK,
    ENBDAPIError
};
```

---

## Q35: How do you implement webhook systems for real-time event notifications to external systems?

**Answer:**

Webhooks enable real-time event notifications by sending HTTP POST requests to client-specified URLs when events occur.

### Webhook Implementation:

```javascript
// webhook-manager.js
const axios = require('axios');
const crypto = require('crypto');
const { Queue } = require('bullmq');

/**
 * Webhook Manager
 */
class WebhookManager {
    constructor(redis) {
        this.redis = redis;
        
        // Create queue for webhook deliveries
        this.webhookQueue = new Queue('webhooks', {
            connection: redis
        });
    }

    /**
     * Register webhook endpoint
     */
    async registerWebhook(webhookData) {
        const {
            clientId,
            url,
            events = [],
            secret = null,
            description = ''
        } = webhookData;

        // Validate URL
        if (!this.isValidURL(url)) {
            throw new Error('Invalid webhook URL');
        }

        // Generate secret if not provided
        const webhookSecret = secret || this.generateSecret();

        // Store webhook configuration
        const result = await db.query(`
            INSERT INTO webhooks (
                client_id,
                url,
                events,
                secret,
                description,
                is_active,
                created_at
            ) VALUES ($1, $2, $3, $4, $5, true, NOW())
            RETURNING id
        `, [
            clientId,
            url,
            JSON.stringify(events),
            this.hashSecret(webhookSecret),
            description
        ]);

        const webhookId = result.rows[0].id;

        await this.logWebhookEvent(webhookId, 'registered', { url, events });

        return {
            webhookId,
            secret: webhookSecret, // Return once for client to store
            url,
            events
        };
    }

    /**
     * Trigger webhook for event
     */
    async triggerWebhook(event) {
        const {
            eventType,
            data,
            customerId = null,
            accountId = null
        } = event;

        // Find all webhooks subscribed to this event
        const webhooks = await this.getWebhooksForEvent(eventType);

        if (webhooks.length === 0) {
            console.log(`No webhooks registered for event: ${eventType}`);
            return;
        }

        // Queue webhook deliveries
        for (const webhook of webhooks) {
            await this.queueWebhookDelivery(webhook, event);
        }
    }

    /**
     * Queue webhook delivery
     */
    async queueWebhookDelivery(webhook, event) {
        const deliveryId = this.generateDeliveryId();

        // Add to queue with retry logic
        await this.webhookQueue.add(
            'deliver',
            {
                deliveryId,
                webhookId: webhook.id,
                url: webhook.url,
                secret: webhook.secret,
                event
            },
            {
                attempts: 5,
                backoff: {
                    type: 'exponential',
                    delay: 2000
                },
                removeOnComplete: {
                    age: 86400, // 24 hours
                    count: 1000
                },
                removeOnFail: false
            }
        );

        console.log(`Queued webhook delivery: ${deliveryId}`);
    }

    /**
     * Deliver webhook
     */
    async deliverWebhook(job) {
        const { deliveryId, webhookId, url, secret, event } = job.data;

        const payload = {
            id: deliveryId,
            event: event.eventType,
            created: new Date().toISOString(),
            data: event.data
        };

        const signature = this.generateSignature(payload, secret);

        const startTime = Date.now();

        try {
            const response = await axios.post(url, payload, {
                headers: {
                    'Content-Type': 'application/json',
                    'X-Webhook-Signature': signature,
                    'X-Webhook-ID': deliveryId,
                    'X-Webhook-Event': event.eventType,
                    'User-Agent': 'ENBD-Webhooks/1.0'
                },
                timeout: 30000,
                validateStatus: (status) => status >= 200 && status < 300
            });

            const duration = Date.now() - startTime;

            // Log successful delivery
            await this.logWebhookDelivery({
                webhookId,
                deliveryId,
                eventType: event.eventType,
                status: 'success',
                statusCode: response.status,
                duration,
                attempt: job.attemptsMade
            });

            console.log(`✅ Webhook delivered: ${deliveryId} (${duration}ms)`);

        } catch (error) {
            const duration = Date.now() - startTime;

            // Log failed delivery
            await this.logWebhookDelivery({
                webhookId,
                deliveryId,
                eventType: event.eventType,
                status: 'failed',
                statusCode: error.response?.status || 0,
                duration,
                attempt: job.attemptsMade,
                error: error.message
            });

            console.error(`❌ Webhook delivery failed: ${deliveryId} - ${error.message}`);

            // Disable webhook after too many failures
            if (job.attemptsMade >= 5) {
                await this.disableWebhook(webhookId, 'Too many delivery failures');
            }

            throw error;
        }
    }

    /**
     * Get webhooks for event type
     */
    async getWebhooksForEvent(eventType) {
        const result = await db.query(`
            SELECT id, url, secret, events
            FROM webhooks
            WHERE is_active = true
            AND ($1 = ANY(string_to_array(events::text, ','))
                 OR events @> '["*"]')
        `, [eventType]);

        return result.rows.map(row => ({
            ...row,
            events: JSON.parse(row.events)
        }));
    }

    /**
     * Verify webhook signature
     */
    verifySignature(payload, signature, secret) {
        const expectedSignature = this.generateSignature(payload, secret);
        return crypto.timingSafeEqual(
            Buffer.from(signature),
            Buffer.from(expectedSignature)
        );
    }

    /**
     * Generate webhook signature
     */
    generateSignature(payload, secret) {
        const payloadString = JSON.stringify(payload);
        return crypto
            .createHmac('sha256', secret)
            .update(payloadString)
            .digest('hex');
    }

    /**
     * Generate webhook secret
     */
    generateSecret() {
        return `whsec_${crypto.randomBytes(32).toString('hex')}`;
    }

    /**
     * Hash secret for storage
     */
    hashSecret(secret) {
        return crypto.createHash('sha256').update(secret).digest('hex');
    }

    /**
     * Disable webhook
     */
    async disableWebhook(webhookId, reason) {
        await db.query(`
            UPDATE webhooks
            SET is_active = false,
                disabled_at = NOW(),
                disabled_reason = $2
            WHERE id = $1
        `, [webhookId, reason]);

        await this.logWebhookEvent(webhookId, 'disabled', { reason });
    }

    /**
     * List webhook deliveries
     */
    async listDeliveries(webhookId, limit = 50) {
        const result = await db.query(`
            SELECT 
                delivery_id,
                event_type,
                status,
                status_code,
                duration,
                attempt,
                error,
                created_at
            FROM webhook_deliveries
            WHERE webhook_id = $1
            ORDER BY created_at DESC
            LIMIT $2
        `, [webhookId, limit]);

        return result.rows;
    }

    /**
     * Retry failed delivery
     */
    async retryDelivery(deliveryId) {
        const result = await db.query(`
            SELECT webhook_id, event_type, event_data
            FROM webhook_deliveries
            WHERE delivery_id = $1
        `, [deliveryId]);

        if (result.rows.length === 0) {
            throw new Error('Delivery not found');
        }

        const delivery = result.rows[0];
        const webhook = await this.getWebhook(delivery.webhook_id);

        await this.queueWebhookDelivery(webhook, {
            eventType: delivery.event_type,
            data: JSON.parse(delivery.event_data)
        });
    }

    async getWebhook(webhookId) {
        const result = await db.query(
            'SELECT * FROM webhooks WHERE id = $1',
            [webhookId]
        );
        return result.rows[0];
    }

    async logWebhookDelivery(delivery) {
        await db.query(`
            INSERT INTO webhook_deliveries (
                webhook_id, delivery_id, event_type, status, status_code,
                duration, attempt, error, created_at
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
        `, [
            delivery.webhookId,
            delivery.deliveryId,
            delivery.eventType,
            delivery.status,
            delivery.statusCode,
            delivery.duration,
            delivery.attempt,
            delivery.error
        ]);
    }

    async logWebhookEvent(webhookId, eventType, metadata) {
        await db.query(`
            INSERT INTO webhook_events (
                webhook_id, event_type, metadata, created_at
            ) VALUES ($1, $2, $3, NOW())
        `, [webhookId, eventType, JSON.stringify(metadata)]);
    }

    generateDeliveryId() {
        return `whd_${Date.now()}_${crypto.randomBytes(8).toString('hex')}`;
    }

    isValidURL(url) {
        try {
            const parsed = new URL(url);
            return ['http:', 'https:'].includes(parsed.protocol);
        } catch {
            return false;
        }
    }
}

/**
 * Webhook Event Types
 */
const WebhookEvents = {
    // Customer events
    CUSTOMER_CREATED: 'customer.created',
    CUSTOMER_UPDATED: 'customer.updated',
    CUSTOMER_DELETED: 'customer.deleted',
    
    // Account events
    ACCOUNT_CREATED: 'account.created',
    ACCOUNT_UPDATED: 'account.updated',
    ACCOUNT_FROZEN: 'account.frozen',
    ACCOUNT_CLOSED: 'account.closed',
    
    // Transaction events
    TRANSACTION_CREATED: 'transaction.created',
    TRANSACTION_COMPLETED: 'transaction.completed',
    TRANSACTION_FAILED: 'transaction.failed',
    
    // Payment events
    PAYMENT_SUCCEEDED: 'payment.succeeded',
    PAYMENT_FAILED: 'payment.failed',
    PAYMENT_REFUNDED: 'payment.refunded'
};

/**
 * Database Schema
 */
const webhookSchema = `
CREATE TABLE webhooks (
    id SERIAL PRIMARY KEY,
    client_id VARCHAR(100) NOT NULL,
    url TEXT NOT NULL,
    events JSONB DEFAULT '[]',
    secret VARCHAR(64) NOT NULL,
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    disabled_at TIMESTAMP,
    disabled_reason TEXT
);

CREATE INDEX idx_webhooks_client ON webhooks(client_id);
CREATE INDEX idx_webhooks_active ON webhooks(is_active) WHERE is_active = true;

CREATE TABLE webhook_deliveries (
    id SERIAL PRIMARY KEY,
    webhook_id INTEGER REFERENCES webhooks(id),
    delivery_id VARCHAR(100) UNIQUE NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB,
    status VARCHAR(20) NOT NULL,
    status_code INTEGER,
    duration INTEGER,
    attempt INTEGER DEFAULT 1,
    error TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_webhook_deliveries_webhook ON webhook_deliveries(webhook_id);
CREATE INDEX idx_webhook_deliveries_created ON webhook_deliveries(created_at);
CREATE INDEX idx_webhook_deliveries_status ON webhook_deliveries(status);

CREATE TABLE webhook_events (
    id SERIAL PRIMARY KEY,
    webhook_id INTEGER REFERENCES webhooks(id),
    event_type VARCHAR(50) NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
`;

// Usage Example
const webhookManager = new WebhookManager(redisClient);

// Trigger webhook when account is created
async function onAccountCreated(account) {
    await webhookManager.triggerWebhook({
        eventType: WebhookEvents.ACCOUNT_CREATED,
        data: {
            accountId: account.id,
            customerId: account.customerId,
            accountType: account.accountType,
            createdAt: account.createdAt
        }
    });
}

module.exports = {
    WebhookManager,
    WebhookEvents
};
```

---
