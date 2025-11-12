# Azure Questions 1-5: Azure Fundamentals & Core Services

---

### Q1. What is Azure and what are its core services relevant to banking applications?

**Answer:**

Microsoft Azure is a comprehensive cloud computing platform offering IaaS, PaaS, and SaaS solutions. For banking applications, key services include compute, storage, databases, AI/ML, security, and compliance.

**Core Azure Services for Banking:**

```javascript
/**
 * Azure Services Overview for ENBD Banking
 */
class AzureBankingServices {
  getCoreServices() {
    return {
      compute: {
        'Azure App Service': {
          use_case: 'Host Node.js banking APIs and web applications',
          features: ['Auto-scaling', 'CI/CD', 'Built-in load balancing', 'SSL/TLS'],
          pricing: 'Pay-per-use, reserved instances available',
          sla: '99.95% availability'
        },
        
        'Azure Functions': {
          use_case: 'Serverless microservices for event-driven operations',
          features: ['Event triggers', 'Auto-scale', 'Pay-per-execution', 'Multiple languages'],
          example: 'Transaction processing, notification triggers, scheduled jobs',
          pricing: 'Consumption plan: first 1M executions free'
        },
        
        'Azure Kubernetes Service (AKS)': {
          use_case: 'Container orchestration for microservices architecture',
          features: ['Managed K8s', 'Auto-scaling', 'CI/CD integration', 'Azure AD integration'],
          example: 'Deploy and manage banking microservices at scale',
          pricing: 'Pay only for VMs, AKS management free'
        },
        
        'Azure Container Instances': {
          use_case: 'Quick container deployment without orchestration',
          features: ['Fast startup', 'Per-second billing', 'No cluster management'],
          example: 'Batch processing, temporary workloads'
        }
      },
      
      storage: {
        'Azure Blob Storage': {
          use_case: 'Store documents, images, backups',
          tiers: {
            hot: 'Frequently accessed data (customer documents)',
            cool: 'Infrequently accessed (30-day minimum, archived statements)',
            archive: 'Rarely accessed (7-year compliance data)'
          },
          features: ['Encryption at rest', 'Versioning', 'Lifecycle management', 'Geo-redundancy'],
          example: 'Store Emirates ID scans, loan documents, bank statements'
        },
        
        'Azure Files': {
          use_case: 'Shared file storage with SMB/NFS protocol',
          features: ['File shares', 'Snapshots', 'Azure AD authentication'],
          example: 'Shared configuration files, legacy app migration'
        },
        
        'Azure Disk Storage': {
          use_case: 'Persistent block storage for VMs',
          types: ['Premium SSD', 'Standard SSD', 'Standard HDD'],
          example: 'Database storage, application volumes'
        }
      },
      
      databases: {
        'Azure Cosmos DB': {
          use_case: 'Globally distributed NoSQL database',
          features: ['Multi-region replication', '<10ms latency', 'Multiple APIs', '99.999% SLA'],
          apis: ['MongoDB', 'Cassandra', 'Gremlin', 'Table', 'SQL'],
          example: 'Customer profiles, session management, real-time transactions',
          pricing: 'Request units (RU/s) + storage'
        },
        
        'Azure SQL Database': {
          use_case: 'Managed relational database (SQL Server)',
          features: ['Auto-tuning', 'Threat detection', 'Backup/restore', 'Point-in-time recovery'],
          tiers: ['Basic', 'Standard', 'Premium', 'Hyperscale'],
          example: 'Core banking data, transactions, customer accounts',
          compliance: 'PCI DSS, ISO 27001, SOC'
        },
        
        'Azure Database for PostgreSQL': {
          use_case: 'Managed PostgreSQL',
          features: ['High availability', 'Automated backups', 'Monitoring'],
          example: 'Analytics, reporting databases'
        },
        
        'Azure Cache for Redis': {
          use_case: 'In-memory caching',
          features: ['Sub-millisecond latency', 'Clustering', 'Persistence'],
          example: 'Session caching, API response caching, rate limiting'
        }
      },
      
      aiml: {
        'Azure OpenAI Service': {
          use_case: 'Enterprise OpenAI models (GPT-4, GPT-3.5, DALL-E)',
          features: ['Private deployment', 'Azure security', 'Compliance', 'SLA guarantees'],
          models: ['GPT-4', 'GPT-3.5-Turbo', 'GPT-4-Turbo', 'DALL-E-3', 'Whisper'],
          example: 'Banking chatbot, document processing, fraud detection',
          compliance: 'Data residency, GDPR compliant'
        },
        
        'Azure Cognitive Services': {
          use_case: 'Pre-built AI APIs',
          services: {
            'Computer Vision': 'OCR, object detection, face recognition',
            'Speech': 'Speech-to-text, text-to-speech, translation',
            'Language': 'Sentiment analysis, entity extraction, translation',
            'Form Recognizer': 'Extract data from forms and documents'
          },
          example: 'Emirates ID verification, check processing, call transcription'
        },
        
        'Azure Machine Learning': {
          use_case: 'Build, train, deploy custom ML models',
          features: ['AutoML', 'MLOps', 'Model registry', 'Deployment endpoints'],
          example: 'Credit scoring, fraud detection, customer churn prediction'
        }
      },
      
      security: {
        'Azure Active Directory (AAD)': {
          use_case: 'Identity and access management',
          features: ['SSO', 'MFA', 'Conditional access', 'B2C for customers'],
          example: 'Employee authentication, customer portal login',
          integration: 'SAML, OAuth 2.0, OpenID Connect'
        },
        
        'Azure Key Vault': {
          use_case: 'Secrets management',
          features: ['Store keys/secrets/certificates', 'HSM-backed', 'Access policies', 'Audit logging'],
          example: 'API keys, database passwords, encryption keys',
          compliance: 'FIPS 140-2 Level 2'
        },
        
        'Azure Security Center': {
          use_case: 'Unified security management',
          features: ['Threat protection', 'Compliance dashboard', 'Recommendations'],
          example: 'Monitor security posture, detect threats'
        },
        
        'Azure DDoS Protection': {
          use_case: 'Protect against DDoS attacks',
          tiers: ['Basic (free)', 'Standard (advanced protection)'],
          example: 'Protect public-facing banking portals'
        }
      },
      
      networking: {
        'Azure Virtual Network': {
          use_case: 'Isolated network in Azure',
          features: ['Subnets', 'NSGs', 'VPN', 'Peering'],
          example: 'Isolate banking services, secure connectivity'
        },
        
        'Azure Application Gateway': {
          use_case: 'Layer 7 load balancer with WAF',
          features: ['SSL termination', 'URL routing', 'WAF', 'Auto-scaling'],
          example: 'Load balance banking APIs, protect from OWASP Top 10'
        },
        
        'Azure Front Door': {
          use_case: 'Global HTTP load balancer',
          features: ['CDN', 'WAF', 'SSL acceleration', 'URL rewriting'],
          example: 'Deliver banking portal globally with low latency'
        },
        
        'Azure VPN Gateway': {
          use_case: 'Secure connection to on-premises',
          features: ['Site-to-site', 'Point-to-site', 'ExpressRoute'],
          example: 'Connect Azure to ENBD data center'
        }
      },
      
      integration: {
        'Azure Service Bus': {
          use_case: 'Enterprise messaging',
          features: ['Queues', 'Topics/subscriptions', 'Transactions', 'Dead-letter queues'],
          example: 'Decouple microservices, async processing'
        },
        
        'Azure Event Grid': {
          use_case: 'Event-driven architecture',
          features: ['Publish-subscribe', 'Event filtering', 'Webhooks'],
          example: 'Trigger actions on account events, notifications'
        },
        
        'Azure API Management': {
          use_case: 'API gateway',
          features: ['Rate limiting', 'Caching', 'Authentication', 'Analytics'],
          example: 'Expose banking APIs securely to partners'
        },
        
        'Azure Logic Apps': {
          use_case: 'Workflow automation',
          features: ['Visual designer', '200+ connectors', 'Enterprise integrations'],
          example: 'Automate loan approval workflow, integrate with CRM'
        }
      },
      
      monitoring: {
        'Azure Monitor': {
          use_case: 'Comprehensive monitoring',
          features: ['Metrics', 'Logs', 'Alerts', 'Dashboards'],
          components: ['Application Insights', 'Log Analytics', 'Metrics Explorer'],
          example: 'Monitor application performance, set alerts'
        },
        
        'Application Insights': {
          use_case: 'APM for applications',
          features: ['Distributed tracing', 'Performance monitoring', 'Exception tracking'],
          example: 'Track API latency, detect errors in banking app'
        }
      },
      
      devops: {
        'Azure DevOps': {
          use_case: 'Complete DevOps platform',
          services: ['Repos', 'Pipelines', 'Boards', 'Artifacts', 'Test Plans'],
          example: 'CI/CD for banking applications, automated testing'
        },
        
        'GitHub Actions': {
          use_case: 'CI/CD from GitHub',
          features: ['Workflow automation', 'Azure integration', 'Marketplace'],
          example: 'Deploy Node.js app to Azure App Service'
        },
        
        'Azure Container Registry': {
          use_case: 'Private Docker registry',
          features: ['Geo-replication', 'Vulnerability scanning', 'Azure AD auth'],
          example: 'Store banking microservice images'
        }
      }
    };
  }
  
  /**
   * Sample Azure architecture for ENBD
   */
  getENBDArchitecture() {
    return {
      frontend: {
        service: 'Azure App Service (Web App)',
        technology: 'React',
        features: ['SSL', 'Custom domain', 'Auto-scale'],
        cdn: 'Azure Front Door for global delivery'
      },
      
      api: {
        service: 'Azure App Service (API App) or AKS',
        technology: 'Node.js + Express',
        features: ['API Management', 'Rate limiting', 'OAuth 2.0'],
        instances: 'Minimum 3 for HA'
      },
      
      microservices: {
        service: 'Azure Kubernetes Service (AKS)',
        services: [
          'Customer Service',
          'Account Service',
          'Transaction Service',
          'Loan Service',
          'Notification Service'
        ],
        scaling: 'Horizontal Pod Autoscaler',
        monitoring: 'Azure Monitor + App Insights'
      },
      
      data: {
        transactional: 'Azure SQL Database (Premium tier)',
        cache: 'Azure Cache for Redis',
        documents: 'Azure Blob Storage (Hot tier)',
        analytics: 'Azure Synapse Analytics',
        nosql: 'Azure Cosmos DB (for sessions, profiles)'
      },
      
      ai: {
        chatbot: 'Azure OpenAI Service (GPT-4)',
        vision: 'Azure Computer Vision (Emirates ID)',
        speech: 'Azure Speech Services',
        custom_ml: 'Azure Machine Learning'
      },
      
      security: {
        identity: 'Azure AD + Azure AD B2C',
        secrets: 'Azure Key Vault',
        waf: 'Azure Application Gateway WAF',
        ddos: 'Azure DDoS Protection Standard',
        encryption: 'Azure Storage Encryption + TDE for SQL'
      },
      
      messaging: {
        async: 'Azure Service Bus',
        events: 'Azure Event Grid',
        streaming: 'Azure Event Hubs'
      },
      
      monitoring: {
        apm: 'Application Insights',
        logs: 'Log Analytics',
        alerts: 'Azure Monitor Alerts',
        dashboard: 'Azure Dashboard + Grafana'
      },
      
      networking: {
        vnet: 'Azure Virtual Network with subnets',
        connectivity: 'Azure ExpressRoute to on-premises',
        dns: 'Azure DNS',
        firewall: 'Azure Firewall'
      },
      
      backup_dr: {
        backup: 'Azure Backup',
        disaster_recovery: 'Azure Site Recovery',
        geo_replication: 'SQL Database geo-replication'
      },
      
      estimated_cost: {
        monthly: 'AED 200,000 - 500,000',
        breakdown: {
          compute: '35%',
          database: '25%',
          storage: '10%',
          ai_services: '15%',
          networking: '10%',
          monitoring: '5%'
        }
      }
    };
  }
}

module.exports = { AzureBankingServices };
```

---

### Q2. How do you deploy a Node.js application to Azure App Service?

**Answer:**

```javascript
/**
 * Azure App Service Deployment for Node.js Banking API
 */
class AzureAppServiceDeployment {
  /**
   * Method 1: Azure CLI Deployment
   */
  async deployWithAzureCLI() {
    const commands = {
      // 1. Login to Azure
      login: 'az login',
      
      // 2. Create resource group
      createResourceGroup: `
        az group create \\
          --name enbd-banking-rg \\
          --location uaenorth
      `,
      
      // 3. Create App Service Plan
      createAppServicePlan: `
        az appservice plan create \\
          --name enbd-api-plan \\
          --resource-group enbd-banking-rg \\
          --sku P1V2 \\
          --is-linux
      `,
      
      // 4. Create Web App
      createWebApp: `
        az webapp create \\
          --name enbd-banking-api \\
          --resource-group enbd-banking-rg \\
          --plan enbd-api-plan \\
          --runtime "NODE|18-lts"
      `,
      
      // 5. Configure environment variables
      setEnvironment: `
        az webapp config appsettings set \\
          --name enbd-banking-api \\
          --resource-group enbd-banking-rg \\
          --settings \\
            NODE_ENV=production \\
            OPENAI_API_KEY=@Microsoft.KeyVault(SecretUri=https://enbd-kv.vault.azure.net/secrets/openai-key/) \\
            DATABASE_URL=@Microsoft.KeyVault(SecretUri=https://enbd-kv.vault.azure.net/secrets/db-url/)
      `,
      
      // 6. Enable HTTPS only
      enableHTTPS: `
        az webapp update \\
          --name enbd-banking-api \\
          --resource-group enbd-banking-rg \\
          --https-only true
      `,
      
      // 7. Deploy from Git
      deployFromGit: `
        az webapp deployment source config \\
          --name enbd-banking-api \\
          --resource-group enbd-banking-rg \\
          --repo-url https://github.com/enbd/banking-api \\
          --branch main \\
          --manual-integration
      `,
      
      // 8. Configure auto-scale
      autoScale: `
        az monitor autoscale create \\
          --resource-group enbd-banking-rg \\
          --resource enbd-banking-api \\
          --resource-type Microsoft.Web/sites \\
          --name enbd-api-autoscale \\
          --min-count 2 \\
          --max-count 10 \\
          --count 3
        
        az monitor autoscale rule create \\
          --resource-group enbd-banking-rg \\
          --autoscale-name enbd-api-autoscale \\
          --condition "CpuPercentage > 70 avg 5m" \\
          --scale out 2
      `
    };
    
    return commands;
  }
  
  /**
   * Method 2: GitHub Actions Deployment
   */
  generateGitHubActionsWorkflow() {
    return `
name: Deploy to Azure App Service

on:
  push:
    branches: [ main ]

env:
  AZURE_WEBAPP_NAME: enbd-banking-api
  AZURE_WEBAPP_PACKAGE_PATH: '.'
  NODE_VERSION: '18.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: \${{ env.NODE_VERSION }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm test
      env:
        NODE_ENV: test
    
    - name: Build application
      run: npm run build --if-present
    
    - name: 'Deploy to Azure WebApp'
      uses: azure/webapps-deploy@v2
      with:
        app-name: \${{ env.AZURE_WEBAPP_NAME }}
        publish-profile: \${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: \${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
    
    - name: Azure logout
      run: |
        az logout
`;
  }
  
  /**
   * Method 3: Docker Container Deployment
   */
  generateDockerDeployment() {
    const dockerfile = `
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

USER node

CMD ["node", "server.js"]
`;
    
    const azureCommands = `
# Build and push to Azure Container Registry
az acr build \\
  --registry enbdregistry \\
  --image banking-api:v1.0 \\
  --file Dockerfile .

# Deploy to App Service from ACR
az webapp create \\
  --resource-group enbd-banking-rg \\
  --plan enbd-api-plan \\
  --name enbd-banking-api \\
  --deployment-container-image-name enbdregistry.azurecr.io/banking-api:v1.0

# Enable continuous deployment
az webapp deployment container config \\
  --name enbd-banking-api \\
  --resource-group enbd-banking-rg \\
  --enable-cd true
`;
    
    return { dockerfile, azureCommands };
  }
  
  /**
   * App Service Configuration
   */
  getAppServiceConfig() {
    return {
      // package.json scripts
      scripts: {
        "start": "node server.js",
        "build": "echo 'No build step required for Node.js'",
        "test": "mocha test/**/*.test.js"
      },
      
      // .deployment file (tells Azure which script to run)
      deployment: `
[config]
SCM_DO_BUILD_DURING_DEPLOYMENT=true
      `.trim(),
      
      // web.config (if needed for IIS)
      webConfig: `
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="iisnode" path="server.js" verb="*" modules="iisnode"/>
    </handlers>
    <rewrite>
      <rules>
        <rule name="NodeInspector" patternSyntax="ECMAScript" stopProcessing="true">
          <match url="^server.js\\/debug[/]?" />
        </rule>
        <rule name="StaticContent">
          <action type="Rewrite" url="public{REQUEST_URI}"/>
        </rule>
        <rule name="DynamicContent">
          <conditions>
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="True"/>
          </conditions>
          <action type="Rewrite" url="server.js"/>
        </rule>
      </rules>
    </rewrite>
    <security>
      <requestFiltering>
        <hiddenSegments>
          <remove segment="bin"/>
        </hiddenSegments>
      </requestFiltering>
    </security>
    <httpErrors existingResponse="PassThrough" />
  </system.webServer>
</configuration>
      `.trim(),
      
      // Application settings (environment variables)
      appSettings: {
        WEBSITE_NODE_DEFAULT_VERSION: '~18',
        NODE_ENV: 'production',
        WEBSITE_RUN_FROM_PACKAGE: '1', // Run from deployment package
        SCM_DO_BUILD_DURING_DEPLOYMENT: 'true',
        WEBSITES_ENABLE_APP_SERVICE_STORAGE: 'false',
        WEBSITES_PORT: '3000' // Custom port if needed
      },
      
      // Startup command
      startupCommand: 'node server.js',
      
      // Health check
      healthCheck: {
        path: '/health',
        interval: 60 // seconds
      }
    };
  }
}

module.exports = { AzureAppServiceDeployment };
```

---

### Q3. How do you implement Azure Functions for serverless banking operations?

**Answer:**

```javascript
/**
 * Azure Functions for Banking Operations
 */

// Function 1: HTTP Trigger - Account Balance API
module.exports = async function (context, req) {
  context.log('Account Balance function triggered');
  
  const customerId = req.query.customerId || (req.body && req.body.customerId);
  
  if (!customerId) {
    context.res = {
      status: 400,
      body: 'Please provide customerId'
    };
    return;
  }
  
  try {
    // Connect to database
    const { Pool } = require('pg');
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL
    });
    
    // Query balance
    const result = await pool.query(
      'SELECT account_number, balance, currency FROM accounts WHERE customer_id = $1',
      [customerId]
    );
    
    if (result.rows.length === 0) {
      context.res = {
        status: 404,
        body: 'Customer not found'
      };
      return;
    }
    
    context.res = {
      status: 200,
      headers: { 'Content-Type': 'application/json' },
      body: {
        customerId,
        accounts: result.rows
      }
    };
  } catch (error) {
    context.log.error('Error:', error);
    context.res = {
      status: 500,
      body: 'Internal server error'
    };
  }
};

// Function 2: Timer Trigger - Daily Transaction Summary
module.exports = async function (context, myTimer) {
  const timeStamp = new Date().toISOString();
  
  context.log('Daily summary function triggered at', timeStamp);
  
  try {
    const { Pool } = require('pg');
    const { QueueClient } = require('@azure/storage-queue');
    
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL
    });
    
    // Get yesterday's transactions
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    
    const result = await pool.query(`
      SELECT 
        customer_id,
        COUNT(*) as transaction_count,
        SUM(amount) as total_amount
      FROM transactions
      WHERE created_at >= $1
      GROUP BY customer_id
    `, [yesterday]);
    
    // Send to queue for email processing
    const queueClient = new QueueClient(
      process.env.STORAGE_CONNECTION_STRING,
      'transaction-summaries'
    );
    
    for (const row of result.rows) {
      await queueClient.sendMessage(JSON.stringify({
        customerId: row.customer_id,
        transactionCount: row.transaction_count,
        totalAmount: row.total_amount,
        date: yesterday.toISOString()
      }));
    }
    
    context.log(`Processed ${result.rows.length} summaries`);
  } catch (error) {
    context.log.error('Error:', error);
    throw error;
  }
};

// Function 3: Queue Trigger - Send Email Notifications
module.exports = async function (context, myQueueItem) {
  context.log('Email notification function triggered');
  context.log('Queue item:', myQueueItem);
  
  const { customerId, transactionCount, totalAmount, date } = myQueueItem;
  
  try {
    // Get customer email
    const { Pool } = require('pg');
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL
    });
    
    const result = await pool.query(
      'SELECT email, name FROM customers WHERE id = $1',
      [customerId]
    );
    
    if (result.rows.length === 0) {
      context.log.warn(`Customer ${customerId} not found`);
      return;
    }
    
    const { email, name } = result.rows[0];
    
    // Send email (using SendGrid or similar)
    const sgMail = require('@sendgrid/mail');
    sgMail.setApiKey(process.env.SENDGRID_API_KEY);
    
    const msg = {
      to: email,
      from: 'noreply@enbd.com',
      subject: 'Your Daily Transaction Summary - Emirates NBD',
      html: `
        <h2>Hello ${name},</h2>
        <p>Here's your transaction summary for ${date}:</p>
        <ul>
          <li>Total Transactions: ${transactionCount}</li>
          <li>Total Amount: AED ${totalAmount.toFixed(2)}</li>
        </ul>
        <p>Thank you for banking with Emirates NBD.</p>
      `
    };
    
    await sgMail.send(msg);
    
    context.log(`Email sent to ${email}`);
  } catch (error) {
    context.log.error('Error sending email:', error);
    throw error;
  }
};

// Function 4: Event Grid Trigger - Fraud Detection
module.exports = async function (context, eventGridEvent) {
  context.log('Fraud detection function triggered');
  context.log('Event:', eventGridEvent);
  
  const { transactionId, customerId, amount, location } = eventGridEvent.data;
  
  try {
    // Simple fraud rules
    const fraudScore = calculateFraudScore({
      amount,
      location,
      time: new Date()
    });
    
    if (fraudScore > 0.7) {
      // High fraud risk - block transaction and alert
      context.log.warn(`High fraud risk detected for transaction ${transactionId}`);
      
      // Update transaction status
      const { Pool } = require('pg');
      const pool = new Pool({
        connectionString: process.env.DATABASE_URL
      });
      
      await pool.query(
        'UPDATE transactions SET status = $1, fraud_score = $2 WHERE id = $3',
        ['blocked', fraudScore, transactionId]
      );
      
      // Send alert to Azure Monitor
      const appInsights = require('applicationinsights');
      appInsights.setup(process.env.APPINSIGHTS_CONNECTION_STRING);
      const client = appInsights.defaultClient;
      
      client.trackEvent({
        name: 'FraudDetected',
        properties: {
          transactionId,
          customerId,
          amount,
          fraudScore
        }
      });
      
      // Notify customer
      await notifyCustomer(customerId, transactionId);
    }
    
    context.log(`Fraud score: ${fraudScore}`);
  } catch (error) {
    context.log.error('Error:', error);
    throw error;
  }
};

function calculateFraudScore(transaction) {
  let score = 0;
  
  // High amount
  if (transaction.amount > 50000) score += 0.3;
  
  // Unusual time (2am - 5am)
  const hour = transaction.time.getHours();
  if (hour >= 2 && hour <= 5) score += 0.2;
  
  // Foreign location (simplified)
  if (transaction.location && !transaction.location.includes('UAE')) {
    score += 0.4;
  }
  
  return Math.min(score, 1);
}

async function notifyCustomer(customerId, transactionId) {
  // Send SMS or push notification
  console.log(`Notifying customer ${customerId} about suspicious transaction ${transactionId}`);
}

/**
 * Function Configuration (function.json)
 */
const functionConfigurations = {
  httpTrigger: {
    "bindings": [
      {
        "authLevel": "function",
        "type": "httpTrigger",
        "direction": "in",
        "name": "req",
        "methods": ["get", "post"]
      },
      {
        "type": "http",
        "direction": "out",
        "name": "res"
      }
    ]
  },
  
  timerTrigger: {
    "bindings": [
      {
        "name": "myTimer",
        "type": "timerTrigger",
        "direction": "in",
        "schedule": "0 0 8 * * *"  // Daily at 8 AM
      }
    ]
  },
  
  queueTrigger: {
    "bindings": [
      {
        "name": "myQueueItem",
        "type": "queueTrigger",
        "direction": "in",
        "queueName": "transaction-summaries",
        "connection": "STORAGE_CONNECTION_STRING"
      }
    ]
  },
  
  eventGridTrigger: {
    "bindings": [
      {
        "type": "eventGridTrigger",
        "name": "eventGridEvent",
        "direction": "in"
      }
    ]
  }
};

module.exports = { functionConfigurations };
```

---

### Q4. How do you configure Azure SQL Database for a banking application?

**Answer:**

```javascript
/**
 * Azure SQL Database Configuration for ENBD Banking
 */
class AzureSQLConfiguration {
  /**
   * Create Azure SQL Database
   */
  async createDatabase() {
    const commands = {
      // 1. Create SQL Server
      createServer: `
        az sql server create \\
          --name enbd-sql-server \\
          --resource-group enbd-banking-rg \\
          --location uaenorth \\
          --admin-user enbdadmin \\
          --admin-password 'ComplexP@ssw0rd!'
      `,
      
      // 2. Configure firewall (allow Azure services)
      configureFirewall: `
        az sql server firewall-rule create \\
          --resource-group enbd-banking-rg \\
          --server enbd-sql-server \\
          --name AllowAzureServices \\
          --start-ip-address 0.0.0.0 \\
          --end-ip-address 0.0.0.0
      `,
      
      // 3. Create database
      createDatabase: `
        az sql db create \\
          --resource-group enbd-banking-rg \\
          --server enbd-sql-server \\
          --name enbd-banking-db \\
          --service-objective P2 \\
          --zone-redundant true \\
          --backup-storage-redundancy GeoZone
      `,
      
      // 4. Enable Transparent Data Encryption
      enableTDE: `
        az sql db tde set \\
          --resource-group enbd-banking-rg \\
          --server enbd-sql-server \\
          --database enbd-banking-db \\
          --status Enabled
      `,
      
      // 5. Configure geo-replication
      geoReplicate: `
        az sql db replica create \\
          --resource-group enbd-banking-rg \\
          --server enbd-sql-server \\
          --name enbd-banking-db \\
          --partner-server enbd-sql-server-secondary \\
          --partner-resource-group enbd-banking-rg-secondary
      `,
      
      // 6. Enable Advanced Threat Protection
      enableThreatProtection: `
        az sql db threat-policy update \\
          --resource-group enbd-banking-rg \\
          --server enbd-sql-server \\
          --name enbd-banking-db \\
          --state Enabled \\
          --email-account-admins Enabled
      `,
      
      // 7. Configure auditing
      configureAuditing: `
        az sql server audit-policy update \\
          --resource-group enbd-banking-rg \\
          --name enbd-sql-server \\
          --state Enabled \\
          --storage-account enbdauditlogs \\
          --retention-days 90
      `
    };
    
    return commands;
  }
  
  /**
   * Database Schema for Banking
   */
  getDatabaseSchema() {
    return `
-- Customers table
CREATE TABLE customers (
    id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    emirates_id NVARCHAR(20) UNIQUE NOT NULL,
    name NVARCHAR(200) NOT NULL,
    email NVARCHAR(255) UNIQUE NOT NULL,
    phone NVARCHAR(20),
    date_of_birth DATE,
    nationality NVARCHAR(50),
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    updated_at DATETIME2 DEFAULT GETUTCDATE(),
    INDEX IX_customers_emirates_id (emirates_id),
    INDEX IX_customers_email (email)
);

-- Accounts table
CREATE TABLE accounts (
    id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    customer_id UNIQUEIDENTIFIER NOT NULL,
    account_number NVARCHAR(20) UNIQUE NOT NULL,
    account_type NVARCHAR(50) NOT NULL CHECK (account_type IN ('savings', 'current', 'fixed_deposit')),
    currency NVARCHAR(3) DEFAULT 'AED',
    balance DECIMAL(18, 2) DEFAULT 0,
    status NVARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'frozen', 'closed')),
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    updated_at DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    INDEX IX_accounts_customer_id (customer_id),
    INDEX IX_accounts_account_number (account_number)
);

-- Transactions table (partitioned by date)
CREATE TABLE transactions (
    id UNIQUEIDENTIFIER DEFAULT NEWID(),
    account_id UNIQUEIDENTIFIER NOT NULL,
    transaction_type NVARCHAR(50) NOT NULL CHECK (transaction_type IN ('debit', 'credit')),
    amount DECIMAL(18, 2) NOT NULL,
    balance_after DECIMAL(18, 2),
    description NVARCHAR(500),
    reference_number NVARCHAR(50) UNIQUE,
    status NVARCHAR(20) DEFAULT 'completed' CHECK (status IN ('pending', 'completed', 'failed', 'blocked')),
    fraud_score DECIMAL(3, 2),
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (account_id) REFERENCES accounts(id)
) ON TransactionsPartitionScheme(created_at);

-- Create partition function (monthly partitions)
CREATE PARTITION FUNCTION TransactionsPartitionFunction (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', '2024-02-01', '2024-03-01', '2024-04-01',
    '2024-05-01', '2024-06-01', '2024-07-01', '2024-08-01',
    '2024-09-01', '2024-10-01', '2024-11-01', '2024-12-01'
);

-- Create partition scheme
CREATE PARTITION SCHEME TransactionsPartitionScheme
AS PARTITION TransactionsPartitionFunction
ALL TO ([PRIMARY]);

-- Loans table
CREATE TABLE loans (
    id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    customer_id UNIQUEIDENTIFIER NOT NULL,
    loan_type NVARCHAR(50) NOT NULL CHECK (loan_type IN ('personal', 'home', 'auto', 'business')),
    amount DECIMAL(18, 2) NOT NULL,
    interest_rate DECIMAL(5, 2) NOT NULL,
    tenure_months INT NOT NULL,
    monthly_payment DECIMAL(18, 2),
    outstanding_balance DECIMAL(18, 2),
    status NVARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'approved', 'disbursed', 'closed', 'rejected')),
    application_date DATETIME2 DEFAULT GETUTCDATE(),
    approval_date DATETIME2,
    disbursement_date DATETIME2,
    FOREIGN KEY (customer_id) REFERENCES customers(id),
    INDEX IX_loans_customer_id (customer_id),
    INDEX IX_loans_status (status)
);

-- Audit log table
CREATE TABLE audit_logs (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    table_name NVARCHAR(100) NOT NULL,
    operation NVARCHAR(20) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    record_id UNIQUEIDENTIFIER,
    old_values NVARCHAR(MAX),
    new_values NVARCHAR(MAX),
    changed_by NVARCHAR(255),
    changed_at DATETIME2 DEFAULT GETUTCDATE(),
    INDEX IX_audit_logs_table_operation (table_name, operation),
    INDEX IX_audit_logs_changed_at (changed_at)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_transactions_created_at 
ON transactions(created_at DESC) 
INCLUDE (account_id, amount, transaction_type);

CREATE NONCLUSTERED INDEX IX_transactions_account_date 
ON transactions(account_id, created_at DESC);

-- Row-level security
CREATE SCHEMA security;
GO

CREATE FUNCTION security.fn_securitypredicate(@customer_id UNIQUEIDENTIFIER)
    RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @customer_id = CAST(SESSION_CONTEXT(N'customer_id') AS UNIQUEIDENTIFIER)
    OR IS_MEMBER('db_owner') = 1;
GO

CREATE SECURITY POLICY security.CustomerFilter
ADD FILTER PREDICATE security.fn_securitypredicate(customer_id)
ON dbo.accounts,
ADD FILTER PREDICATE security.fn_securitypredicate(customer_id)
ON dbo.loans
WITH (STATE = ON);
GO

-- Dynamic Data Masking
ALTER TABLE customers
ALTER COLUMN email ADD MASKED WITH (FUNCTION = 'email()');

ALTER TABLE customers
ALTER COLUMN phone ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)');

ALTER TABLE customers
ALTER COLUMN emirates_id ADD MASKED WITH (FUNCTION = 'partial(3,"XXX-XXXX-XXXXX-",1)');
`;
  }
  
  /**
   * Node.js connection with Azure SQL
   */
  getNodeJSConnection() {
    return `
const sql = require('mssql');
const { DefaultAzureCredential } = require('@azure/identity');

class AzureSQLService {
  constructor() {
    this.config = {
      server: process.env.SQL_SERVER,
      database: process.env.SQL_DATABASE,
      options: {
        encrypt: true,
        trustServerCertificate: false,
        enableArithAbort: true
      },
      // Option 1: SQL Authentication
      user: process.env.SQL_USER,
      password: process.env.SQL_PASSWORD,
      
      // Option 2: Azure AD Authentication
      authentication: {
        type: 'azure-active-directory-default'
      }
    };
  }
  
  async connect() {
    this.pool = await sql.connect(this.config);
    console.log('Connected to Azure SQL Database');
  }
  
  async getCustomer(customerId) {
    const result = await this.pool.request()
      .input('customerId', sql.UniqueIdentifier, customerId)
      .query('SELECT * FROM customers WHERE id = @customerId');
    
    return result.recordset[0];
  }
  
  async createTransaction(accountId, amount, type) {
    const transaction = new sql.Transaction(this.pool);
    
    try {
      await transaction.begin();
      
      // Insert transaction
      const result = await transaction.request()
        .input('accountId', sql.UniqueIdentifier, accountId)
        .input('amount', sql.Decimal(18, 2), amount)
        .input('type', sql.NVarChar(50), type)
        .query(\`
          INSERT INTO transactions (account_id, amount, transaction_type)
          OUTPUT INSERTED.id
          VALUES (@accountId, @amount, @type)
        \`);
      
      // Update account balance
      await transaction.request()
        .input('accountId', sql.UniqueIdentifier, accountId)
        .input('amount', sql.Decimal(18, 2), amount)
        .input('type', sql.NVarChar(50), type)
        .query(\`
          UPDATE accounts
          SET balance = balance + CASE WHEN @type = 'credit' THEN @amount ELSE -@amount END,
              updated_at = GETUTCDATE()
          WHERE id = @accountId
        \`);
      
      await transaction.commit();
      
      return result.recordset[0];
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
  
  async close() {
    await this.pool.close();
  }
}

module.exports = { AzureSQLService };
`;
  }
}

module.exports = { AzureSQLConfiguration };
```

---

### Q5. How do you implement Azure Key Vault for secrets management?

**Answer:**

```javascript
/**
 * Azure Key Vault Integration for Banking Secrets
 */
const { SecretClient } = require('@azure/keyvault-secrets');
const { DefaultAzureCredential } = require('@azure/identity');

class AzureKeyVaultService {
  constructor() {
    const vaultName = process.env.KEY_VAULT_NAME || 'enbd-banking-kv';
    const vaultUrl = `https://${vaultName}.vault.azure.net`;
    
    // Use Azure AD authentication (Managed Identity in production)
    this.credential = new DefaultAzureCredential();
    this.secretClient = new SecretClient(vaultUrl, this.credential);
  }
  
  /**
   * Get secret from Key Vault
   */
  async getSecret(secretName) {
    try {
      const secret = await this.secretClient.getSecret(secretName);
      return secret.value;
    } catch (error) {
      console.error(`Error retrieving secret ${secretName}:`, error.message);
      throw error;
    }
  }
  
  /**
   * Set secret in Key Vault
   */
  async setSecret(secretName, secretValue, options = {}) {
    try {
      const secret = await this.secretClient.setSecret(secretName, secretValue, {
        contentType: options.contentType || 'text/plain',
        enabled: true,
        tags: options.tags || {},
        expiresOn: options.expiresOn // Optional expiration date
      });
      
      return {
        name: secret.name,
        version: secret.properties.version,
        createdOn: secret.properties.createdOn
      };
    } catch (error) {
      console.error(`Error setting secret ${secretName}:`, error.message);
      throw error;
    }
  }
  
  /**
   * List all secrets
   */
  async listSecrets() {
    const secrets = [];
    
    for await (const properties of this.secretClient.listPropertiesOfSecrets()) {
      secrets.push({
        name: properties.name,
        enabled: properties.enabled,
        createdOn: properties.createdOn,
        updatedOn: properties.updatedOn,
        expiresOn: properties.expiresOn
      });
    }
    
    return secrets;
  }
  
  /**
   * Delete secret
   */
  async deleteSecret(secretName) {
    try {
      const poller = await this.secretClient.beginDeleteSecret(secretName);
      await poller.pollUntilDone();
      
      return { deleted: true, secretName };
    } catch (error) {
      console.error(`Error deleting secret ${secretName}:`, error.message);
      throw error;
    }
  }
  
  /**
   * Rotate secret (create new version)
   */
  async rotateSecret(secretName, newValue) {
    // Set new version
    const result = await this.setSecret(secretName, newValue, {
      tags: { rotated: new Date().toISOString() }
    });
    
    console.log(`Secret ${secretName} rotated. New version: ${result.version}`);
    
    return result;
  }
}

/**
 * Banking Secrets Manager
 */
class BankingSecretsManager {
  constructor() {
    this.keyVault = new AzureKeyVaultService();
    this.cache = new Map();
    this.cacheTTL = 300000; // 5 minutes
  }
  
  /**
   * Get database connection string
   */
  async getDatabaseConnectionString() {
    return await this.getCachedSecret('database-connection-string');
  }
  
  /**
   * Get OpenAI API key
   */
  async getOpenAIKey() {
    return await this.getCachedSecret('openai-api-key');
  }
  
  /**
   * Get Redis connection string
   */
  async getRedisConnectionString() {
    return await this.getCachedSecret('redis-connection-string');
  }
  
  /**
   * Get JWT secret
   */
  async getJWTSecret() {
    return await this.getCachedSecret('jwt-secret');
  }
  
  /**
   * Get encryption key
   */
  async getEncryptionKey() {
    return await this.getCachedSecret('encryption-key');
  }
  
  /**
   * Cache secret to reduce Key Vault calls
   */
  async getCachedSecret(secretName) {
    const cached = this.cache.get(secretName);
    
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached.value;
    }
    
    // Fetch from Key Vault
    const value = await this.keyVault.getSecret(secretName);
    
    // Cache it
    this.cache.set(secretName, {
      value,
      timestamp: Date.now()
    });
    
    return value;
  }
  
  /**
   * Initialize all secrets
   */
  async initializeSecrets() {
    const secrets = {
      'database-connection-string': process.env.DATABASE_URL,
      'openai-api-key': process.env.OPENAI_API_KEY,
      'redis-connection-string': process.env.REDIS_URL,
      'jwt-secret': this.generateRandomSecret(64),
      'encryption-key': this.generateRandomSecret(32)
    };
    
    for (const [name, value] of Object.entries(secrets)) {
      if (value) {
        await this.keyVault.setSecret(name, value, {
          tags: { environment: 'production', created_by: 'admin' }
        });
        console.log(`Secret ${name} initialized`);
      }
    }
  }
  
  generateRandomSecret(length) {
    const crypto = require('crypto');
    return crypto.randomBytes(length).toString('hex');
  }
  
  /**
   * Rotate all secrets (scheduled operation)
   */
  async rotateAllSecrets() {
    const secretsToRotate = [
      'jwt-secret',
      'encryption-key'
    ];
    
    for (const secretName of secretsToRotate) {
      const newValue = this.generateRandomSecret(64);
      await this.keyVault.rotateSecret(secretName, newValue);
      
      // Clear cache
      this.cache.delete(secretName);
      
      console.log(`Secret ${secretName} rotated`);
    }
  }
}

/**
 * Use in application
 */
async function initializeApp() {
  const secretsManager = new BankingSecretsManager();
  
  // Get secrets
  const dbConnection = await secretsManager.getDatabaseConnectionString();
  const openaiKey = await secretsManager.getOpenAIKey();
  const jwtSecret = await secretsManager.getJWTSecret();
  
  // Use them in your app
  const app = require('express')();
  
  app.get('/health', (req, res) => {
    res.json({ status: 'healthy', vault: 'connected' });
  });
  
  app.listen(3000);
}

module.exports = { AzureKeyVaultService, BankingSecretsManager };
```

**Azure CLI Commands for Key Vault:**

```bash
# Create Key Vault
az keyvault create \
  --name enbd-banking-kv \
  --resource-group enbd-banking-rg \
  --location uaenorth \
  --enable-rbac-authorization false \
  --enable-soft-delete true \
  --soft-delete-retention-days 90

# Set access policy for App Service
az keyvault set-policy \
  --name enbd-banking-kv \
  --object-id <app-service-managed-identity-object-id> \
  --secret-permissions get list

# Add secret
az keyvault secret set \
  --vault-name enbd-banking-kv \
  --name openai-api-key \
  --value "sk-..."

# Get secret
az keyvault secret show \
  --vault-name enbd-banking-kv \
  --name openai-api-key \
  --query "value" -o tsv

# Enable diagnostic logging
az monitor diagnostic-settings create \
  --resource "/subscriptions/.../resourceGroups/enbd-banking-rg/providers/Microsoft.KeyVault/vaults/enbd-banking-kv" \
  --name KeyVaultLogs \
  --workspace "/subscriptions/.../resourceGroups/enbd-banking-rg/providers/Microsoft.OperationalInsights/workspaces/enbd-logs" \
  --logs '[{"category": "AuditEvent", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

---

**Summary Q1-Q5:**
- Azure core services for banking (compute, storage, databases, AI, security) ✅
- Deploy Node.js to Azure App Service with CI/CD ✅
- Azure Functions for serverless banking operations ✅
- Azure SQL Database configuration with security ✅
- Azure Key Vault for secrets management ✅
