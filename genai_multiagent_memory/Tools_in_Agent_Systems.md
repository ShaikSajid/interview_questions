# Tools in Agent Systems - Complete Guide

## Overview

Tools are external functions that agents can call to interact with the outside world. They enable agents to perform actions beyond text generation - like searching databases, calling APIs, executing code, or accessing external services.

---

## Table of Contents

1. [What are Tools?](#what-are-tools)
2. [Tool Types and Categories](#tool-types-and-categories)
3. [Built-in LangChain Tools](#built-in-langchain-tools)
4. [Creating Custom Tools](#creating-custom-tools)
5. [Tool Calling in OpenAI](#tool-calling-in-openai)
6. [Tools in Multi-Agent Systems](#tools-in-multi-agent-systems)
7. [ENBD Banking Tools Examples](#enbd-banking-tools-examples)
8. [Best Practices](#best-practices)

---

## What are Tools?

### Concept

**Tool** = A function that an agent can call to perform a specific action.

```
┌──────────────┐
│    Agent     │
│   (LLM)      │
└──────┬───────┘
       │ "I need to search customer data"
       ▼
┌──────────────┐
│    Tool      │
│  (Function)  │
└──────┬───────┘
       │ Execute search
       ▼
┌──────────────┐
│   Database   │
└──────────────┘
```

### Why Tools?

✅ **Extend LLM capabilities** beyond text generation  
✅ **Access real-time data** (APIs, databases)  
✅ **Perform actions** (send emails, create tickets)  
✅ **Execute computations** (complex calculations)  
✅ **Integrate systems** (CRM, banking core, etc.)

---

## Tool Types and Categories

### 1. Information Retrieval Tools

**Purpose**: Fetch data from external sources

```typescript
// Database Query Tool
const queryCustomerTool = {
  name: 'query_customer_database',
  description: 'Search customer information in ENBD database',
  parameters: {
    type: 'object',
    properties: {
      customer_id: {
        type: 'string',
        description: 'Customer ID (e.g., CUST-12345)'
      },
      query_type: {
        type: 'string',
        enum: ['profile', 'transactions', 'accounts'],
        description: 'Type of information to retrieve'
      }
    },
    required: ['customer_id', 'query_type']
  }
};

// Implementation
async function queryCustomer(customerId: string, queryType: string) {
  // Database query logic
  const db = await connectToDatabase();
  const result = await db.query(
    `SELECT * FROM customers WHERE id = $1`,
    [customerId]
  );
  return result;
}
```

### 2. Action Tools

**Purpose**: Perform operations or trigger events

```typescript
// Send Email Tool
const sendEmailTool = {
  name: 'send_email',
  description: 'Send email notification to customer',
  parameters: {
    type: 'object',
    properties: {
      recipient: {
        type: 'string',
        description: 'Email address'
      },
      subject: {
        type: 'string',
        description: 'Email subject'
      },
      body: {
        type: 'string',
        description: 'Email body content'
      }
    },
    required: ['recipient', 'subject', 'body']
  }
};

// Implementation
async function sendEmail(recipient: string, subject: string, body: string) {
  const emailService = new SendGridService();
  await emailService.send({
    to: recipient,
    subject,
    html: body
  });
  return { status: 'sent', timestamp: new Date() };
}
```

### 3. Computation Tools

**Purpose**: Perform calculations or data processing

```typescript
// Calculate Loan EMI Tool
const calculateEMITool = {
  name: 'calculate_loan_emi',
  description: 'Calculate monthly EMI for a loan',
  parameters: {
    type: 'object',
    properties: {
      principal: {
        type: 'number',
        description: 'Loan amount in AED'
      },
      rate: {
        type: 'number',
        description: 'Annual interest rate (e.g., 5.5 for 5.5%)'
      },
      tenure: {
        type: 'number',
        description: 'Loan tenure in months'
      }
    },
    required: ['principal', 'rate', 'tenure']
  }
};

// Implementation
function calculateEMI(principal: number, rate: number, tenure: number): number {
  const monthlyRate = rate / 12 / 100;
  const emi = (principal * monthlyRate * Math.pow(1 + monthlyRate, tenure)) /
              (Math.pow(1 + monthlyRate, tenure) - 1);
  return Math.round(emi * 100) / 100;
}
```

### 4. Verification Tools

**Purpose**: Validate or verify information

```typescript
// Emirates ID Verification Tool
const verifyEmiratesIDTool = {
  name: 'verify_emirates_id',
  description: 'Verify Emirates ID with ICP (Identity and Citizenship Platform)',
  parameters: {
    type: 'object',
    properties: {
      id_number: {
        type: 'string',
        description: 'Emirates ID number (format: 784-YYYY-NNNNNNN-N)'
      },
      date_of_birth: {
        type: 'string',
        description: 'Date of birth (YYYY-MM-DD)'
      }
    },
    required: ['id_number', 'date_of_birth']
  }
};

// Implementation
async function verifyEmiratesID(idNumber: string, dob: string) {
  const icpApi = new ICPApiClient(process.env.ICP_API_KEY);
  const result = await icpApi.verify({
    emiratesId: idNumber,
    dateOfBirth: dob
  });
  
  return {
    valid: result.isValid,
    name: result.fullName,
    expiryDate: result.expiryDate,
    confidence: result.confidence
  };
}
```

---

## Built-in LangChain Tools

### 1. Search Tools

```typescript
import { SerpAPI } from '@langchain/community/tools/serpapi';
import { Calculator } from '@langchain/community/tools/calculator';

// Google Search
const searchTool = new SerpAPI(process.env.SERPAPI_API_KEY);

// Usage in agent
const result = await searchTool.call('UAE Central Bank interest rates 2025');
```

### 2. Calculator

```typescript
import { Calculator } from '@langchain/community/tools/calculator';

const calculator = new Calculator();

// Usage
const result = await calculator.call('(500000 * 0.055) / 12');
// Returns: "2291.67"
```

### 3. Retrieval Tools

```typescript
import { Retriever } from '@langchain/core/retrievers';
import { VectorStoreRetriever } from '@langchain/core/vectorstores';

// Vector Store Retriever
const retriever = vectorStore.asRetriever({
  k: 5,  // Return top 5 results
  searchType: 'similarity'
});

// Usage
const docs = await retriever.getRelevantDocuments(
  'What are ENBD home loan requirements?'
);
```

### 4. Request Tools

```typescript
import { RequestsGetTool, RequestsPostTool } from '@langchain/community/tools/requests';

// HTTP GET
const getTool = new RequestsGetTool();
const response = await getTool.call('https://api.enbd.com/rates');

// HTTP POST
const postTool = new RequestsPostTool();
const result = await postTool.call(
  JSON.stringify({
    url: 'https://api.enbd.com/loans',
    data: { amount: 500000, tenure: 60 }
  })
);
```

---

## Creating Custom Tools

### Method 1: Using DynamicStructuredTool

```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';
import { z } from 'zod';

// Define tool with Zod schema
const customerLookupTool = new DynamicStructuredTool({
  name: 'customer_lookup',
  description: 'Look up customer information in ENBD CRM system',
  schema: z.object({
    customerId: z.string().describe('Customer ID (e.g., CUST-12345)'),
    fields: z.array(z.string()).optional().describe('Specific fields to return')
  }),
  func: async ({ customerId, fields }) => {
    console.log(`Looking up customer: ${customerId}`);
    
    // Simulate database query
    const customer = await db.customers.findById(customerId);
    
    if (!customer) {
      return `Customer ${customerId} not found`;
    }
    
    if (fields && fields.length > 0) {
      const filtered = {};
      fields.forEach(field => {
        if (customer[field]) {
          filtered[field] = customer[field];
        }
      });
      return JSON.stringify(filtered);
    }
    
    return JSON.stringify(customer);
  }
});
```

### Method 2: Using Tool Decorator (Class-based)

```typescript
import { Tool } from '@langchain/core/tools';

class CreditScoreTool extends Tool {
  name = 'credit_score_checker';
  
  description = `Check credit score for a customer. 
  Input should be customer ID as a string.
  Returns credit score (300-850) and rating.`;

  async _call(customerId: string): Promise<string> {
    try {
      // Call credit bureau API
      const bureauApi = new CreditBureauAPI();
      const score = await bureauApi.getScore(customerId);
      
      let rating: string;
      if (score >= 750) rating = 'Excellent';
      else if (score >= 700) rating = 'Good';
      else if (score >= 650) rating = 'Fair';
      else rating = 'Poor';
      
      return JSON.stringify({
        customerId,
        creditScore: score,
        rating,
        timestamp: new Date().toISOString()
      });
    } catch (error) {
      return `Error checking credit score: ${error.message}`;
    }
  }
}

// Usage
const creditTool = new CreditScoreTool();
const result = await creditTool.call('CUST-12345');
```

### Method 3: Simple Function Tool

```typescript
import { DynamicTool } from '@langchain/core/tools';

// Simple tool without schema
const getCurrentTimeTool = new DynamicTool({
  name: 'get_current_time',
  description: 'Returns current date and time in UAE timezone',
  func: async () => {
    const now = new Date();
    const uaeTime = now.toLocaleString('en-AE', {
      timeZone: 'Asia/Dubai',
      dateStyle: 'full',
      timeStyle: 'long'
    });
    return uaeTime;
  }
});
```

---

## Tool Calling in OpenAI

### Function Calling API

```typescript
import { ChatOpenAI } from '@langchain/openai';

const model = new ChatOpenAI({
  modelName: 'gpt-4',
  temperature: 0
});

// Define tools/functions
const tools = [
  {
    type: 'function',
    function: {
      name: 'get_customer_balance',
      description: 'Get current account balance for a customer',
      parameters: {
        type: 'object',
        properties: {
          customer_id: {
            type: 'string',
            description: 'Customer ID'
          },
          account_type: {
            type: 'string',
            enum: ['savings', 'current', 'credit'],
            description: 'Type of account'
          }
        },
        required: ['customer_id', 'account_type']
      }
    }
  }
];

// Bind tools to model
const modelWithTools = model.bind({ tools });

// Invoke
const response = await modelWithTools.invoke(
  'What is the savings account balance for customer CUST-12345?'
);

// Check if model wants to call a tool
if (response.tool_calls && response.tool_calls.length > 0) {
  const toolCall = response.tool_calls[0];
  
  console.log('Tool:', toolCall.name);
  console.log('Arguments:', toolCall.args);
  
  // Execute the tool
  if (toolCall.name === 'get_customer_balance') {
    const result = await getCustomerBalance(
      toolCall.args.customer_id,
      toolCall.args.account_type
    );
    
    console.log('Result:', result);
  }
}
```

### Tool Calling Flow

```
┌─────────────────────────────────────────────────────────┐
│  1. User Query                                          │
│  "What's my account balance?"                           │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  2. LLM Analyzes Query + Available Tools                │
│  - Identifies need for get_customer_balance tool        │
│  - Extracts parameters from context                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  3. LLM Returns Tool Call                               │
│  {                                                      │
│    "name": "get_customer_balance",                      │
│    "arguments": {                                       │
│      "customer_id": "CUST-12345",                       │
│      "account_type": "savings"                          │
│    }                                                    │
│  }                                                      │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  4. Application Executes Tool                           │
│  const balance = await getCustomerBalance(...)          │
│  Returns: { balance: 125430.50, currency: "AED" }      │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  5. Send Tool Result Back to LLM                        │
│  "Tool result: AED 125,430.50"                          │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  6. LLM Generates Final Response                        │
│  "Your savings account balance is AED 125,430.50"       │
└─────────────────────────────────────────────────────────┘
```

---

## Tools in Multi-Agent Systems

### Agent-Specific Tools

Each agent gets specialized tools for their role:

```typescript
// Fraud Detection Agent Tools
const fraudAgentTools = [
  new DynamicStructuredTool({
    name: 'check_transaction_pattern',
    description: 'Analyze transaction patterns for anomalies',
    schema: z.object({
      customerId: z.string(),
      days: z.number().default(30)
    }),
    func: async ({ customerId, days }) => {
      // ML-based pattern analysis
      const patterns = await analyzePatterns(customerId, days);
      return JSON.stringify(patterns);
    }
  }),
  
  new DynamicStructuredTool({
    name: 'get_fraud_score',
    description: 'Calculate fraud risk score for a transaction',
    schema: z.object({
      transactionId: z.string(),
      amount: z.number(),
      merchant: z.string()
    }),
    func: async ({ transactionId, amount, merchant }) => {
      const score = await calculateFraudScore(transactionId, amount, merchant);
      return JSON.stringify({ fraudScore: score, risk: score > 70 ? 'HIGH' : 'LOW' });
    }
  })
];

// Credit Assessment Agent Tools
const creditAgentTools = [
  new CreditScoreTool(),
  
  new DynamicStructuredTool({
    name: 'calculate_dti',
    description: 'Calculate debt-to-income ratio',
    schema: z.object({
      monthlyIncome: z.number(),
      monthlyDebts: z.number()
    }),
    func: async ({ monthlyIncome, monthlyDebts }) => {
      const dti = (monthlyDebts / monthlyIncome) * 100;
      return JSON.stringify({ dtiRatio: dti.toFixed(2) });
    }
  })
];

// KYC Verification Agent Tools
const kycAgentTools = [
  new DynamicStructuredTool({
    name: 'verify_emirates_id',
    description: 'Verify Emirates ID with ICP',
    schema: z.object({
      idNumber: z.string(),
      dob: z.string()
    }),
    func: async ({ idNumber, dob }) => {
      const result = await icpVerification(idNumber, dob);
      return JSON.stringify(result);
    }
  }),
  
  new DynamicStructuredTool({
    name: 'check_sanctions_list',
    description: 'Check if customer is on sanctions list',
    schema: z.object({
      name: z.string(),
      nationality: z.string()
    }),
    func: async ({ name, nationality }) => {
      const sanctioned = await checkSanctions(name, nationality);
      return JSON.stringify({ onSanctionsList: sanctioned });
    }
  })
];
```

### Tool Routing in Hierarchical Systems

```typescript
class ManagerAgent {
  private workerTools: Map<string, Tool[]> = new Map();
  
  constructor() {
    // Register worker-specific tools
    this.workerTools.set('fraud_detection', fraudAgentTools);
    this.workerTools.set('credit_assessment', creditAgentTools);
    this.workerTools.set('kyc_verification', kycAgentTools);
  }
  
  async delegateWithTools(taskType: string, task: string) {
    const tools = this.workerTools.get(taskType);
    
    if (!tools) {
      throw new Error(`No tools available for task type: ${taskType}`);
    }
    
    // Create worker agent with specific tools
    const worker = await createAgent({
      llm: this.model,
      tools,
      systemMessage: `You are a ${taskType} specialist.`
    });
    
    const result = await worker.invoke(task);
    return result;
  }
}
```

---

## ENBD Banking Tools Examples

### Complete Tool Suite for Loan Processing

```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';
import { z } from 'zod';

// 1. Customer Information Tool
const getCustomerInfoTool = new DynamicStructuredTool({
  name: 'get_customer_info',
  description: 'Retrieve customer profile from ENBD CRM',
  schema: z.object({
    customerId: z.string().describe('Customer ID')
  }),
  func: async ({ customerId }) => {
    const db = await connectToDatabase();
    const customer = await db.query(
      'SELECT * FROM customers WHERE customer_id = $1',
      [customerId]
    );
    
    return JSON.stringify({
      id: customer.id,
      name: customer.full_name,
      email: customer.email,
      phone: customer.phone,
      accountType: customer.account_type,
      relationship: customer.relationship_manager,
      joinDate: customer.join_date
    });
  }
});

// 2. Credit Bureau Integration
const checkCreditBureauTool = new DynamicStructuredTool({
  name: 'check_credit_bureau',
  description: 'Check credit report from UAE Credit Bureau (Al Etihad)',
  schema: z.object({
    emiratesId: z.string().describe('Emirates ID'),
    customerId: z.string().describe('Customer ID')
  }),
  func: async ({ emiratesId, customerId }) => {
    const bureauApi = new UAECreditBureauAPI({
      apiKey: process.env.CREDIT_BUREAU_API_KEY
    });
    
    const report = await bureauApi.getCreditReport({
      emiratesId,
      customerId,
      reportType: 'full'
    });
    
    return JSON.stringify({
      creditScore: report.score,
      rating: report.rating,
      totalDebt: report.totalDebt,
      activeLoans: report.activeLoans.length,
      paymentHistory: report.paymentHistory,
      inquiries: report.recentInquiries,
      recommendation: report.score >= 700 ? 'APPROVE' : 'REVIEW'
    });
  }
});

// 3. Salary Verification (Direct Integration with Employers)
const verifySalaryTool = new DynamicStructuredTool({
  name: 'verify_salary',
  description: 'Verify salary with WPS (Wage Protection System)',
  schema: z.object({
    emiratesId: z.string().describe('Emirates ID'),
    employerId: z.string().describe('Employer ID')
  }),
  func: async ({ emiratesId, employerId }) => {
    const wpsApi = new WPSIntegrationAPI({
      apiKey: process.env.WPS_API_KEY
    });
    
    const salaryData = await wpsApi.verifySalary({
      emiratesId,
      employerId,
      months: 6  // Last 6 months
    });
    
    return JSON.stringify({
      verified: salaryData.verified,
      averageSalary: salaryData.averageMonthlySalary,
      lastSalaryDate: salaryData.lastPaymentDate,
      employer: salaryData.employerName,
      consistency: salaryData.consistencyScore,
      employmentDuration: salaryData.employmentMonths
    });
  }
});

// 4. Property Valuation Tool
const getPropertyValuationTool = new DynamicStructuredTool({
  name: 'get_property_valuation',
  description: 'Get property valuation from Dubai Land Department',
  schema: z.object({
    propertyId: z.string().describe('Property registration number'),
    location: z.string().describe('Property location')
  }),
  func: async ({ propertyId, location }) => {
    const dldApi = new DubaiLandDepartmentAPI({
      apiKey: process.env.DLD_API_KEY
    });
    
    const valuation = await dldApi.getPropertyValue({
      propertyId,
      location
    });
    
    return JSON.stringify({
      currentValue: valuation.marketValue,
      lastTransactionValue: valuation.lastSalePrice,
      lastTransactionDate: valuation.lastSaleDate,
      propertyType: valuation.type,
      area: valuation.area,
      pricePerSqFt: valuation.pricePerSqFt,
      trend: valuation.priceTrend
    });
  }
});

// 5. Loan Calculator Tool
const calculateLoanTool = new DynamicStructuredTool({
  name: 'calculate_loan_details',
  description: 'Calculate loan EMI, interest, and total payment',
  schema: z.object({
    principal: z.number().describe('Loan amount in AED'),
    rate: z.number().describe('Annual interest rate (e.g., 5.5)'),
    tenure: z.number().describe('Loan tenure in years'),
    loanType: z.enum(['home', 'personal', 'auto']).describe('Type of loan')
  }),
  func: async ({ principal, rate, tenure, loanType }) => {
    // Apply bank-specific rates
    let actualRate = rate;
    if (loanType === 'home') {
      actualRate = rate - 0.5;  // Home loans get 0.5% discount
    }
    
    const monthlyRate = actualRate / 12 / 100;
    const months = tenure * 12;
    
    const emi = (principal * monthlyRate * Math.pow(1 + monthlyRate, months)) /
                (Math.pow(1 + monthlyRate, months) - 1);
    
    const totalPayment = emi * months;
    const totalInterest = totalPayment - principal;
    
    return JSON.stringify({
      loanAmount: principal,
      interestRate: actualRate,
      tenure: tenure,
      monthlyEMI: Math.round(emi * 100) / 100,
      totalInterest: Math.round(totalInterest * 100) / 100,
      totalPayment: Math.round(totalPayment * 100) / 100,
      loanType
    });
  }
});

// 6. Risk Assessment Tool
const assessLoanRiskTool = new DynamicStructuredTool({
  name: 'assess_loan_risk',
  description: 'Assess risk level for loan application',
  schema: z.object({
    creditScore: z.number(),
    dtiRatio: z.number().describe('Debt-to-income ratio'),
    ltvRatio: z.number().describe('Loan-to-value ratio'),
    employmentYears: z.number(),
    existingLoans: z.number()
  }),
  func: async ({ creditScore, dtiRatio, ltvRatio, employmentYears, existingLoans }) => {
    let riskScore = 0;
    
    // Credit score impact (40% weight)
    if (creditScore < 650) riskScore += 40;
    else if (creditScore < 700) riskScore += 25;
    else if (creditScore < 750) riskScore += 10;
    
    // DTI impact (30% weight)
    if (dtiRatio > 50) riskScore += 30;
    else if (dtiRatio > 40) riskScore += 20;
    else if (dtiRatio > 30) riskScore += 10;
    
    // LTV impact (20% weight)
    if (ltvRatio > 85) riskScore += 20;
    else if (ltvRatio > 75) riskScore += 12;
    else if (ltvRatio > 65) riskScore += 5;
    
    // Employment stability (10% weight)
    if (employmentYears < 1) riskScore += 10;
    else if (employmentYears < 3) riskScore += 5;
    
    // Existing loans impact
    if (existingLoans > 3) riskScore += 5;
    
    let riskLevel: string;
    if (riskScore < 20) riskLevel = 'LOW';
    else if (riskScore < 40) riskLevel = 'MEDIUM';
    else riskLevel = 'HIGH';
    
    return JSON.stringify({
      riskScore,
      riskLevel,
      recommendation: riskScore < 30 ? 'APPROVE' : 'REVIEW',
      factors: {
        creditScore,
        dtiRatio,
        ltvRatio,
        employmentYears,
        existingLoans
      }
    });
  }
});

// 7. Compliance Check Tool
const checkComplianceTool = new DynamicStructuredTool({
  name: 'check_compliance',
  description: 'Check AML/KYC compliance and sanctions',
  schema: z.object({
    customerId: z.string(),
    customerName: z.string(),
    nationality: z.string()
  }),
  func: async ({ customerId, customerName, nationality }) => {
    // Check multiple compliance databases
    const [sanctionsResult, pepResult, amlResult] = await Promise.all([
      checkSanctionsList(customerName, nationality),
      checkPEPDatabase(customerName, nationality),
      performAMLCheck(customerId)
    ]);
    
    const allClear = !sanctionsResult.match && 
                     !pepResult.isPEP && 
                     amlResult.riskLevel === 'LOW';
    
    return JSON.stringify({
      compliant: allClear,
      sanctions: {
        onList: sanctionsResult.match,
        lists: sanctionsResult.matchedLists
      },
      pep: {
        isPEP: pepResult.isPEP,
        position: pepResult.position
      },
      aml: {
        riskLevel: amlResult.riskLevel,
        score: amlResult.score
      },
      recommendation: allClear ? 'PROCEED' : 'MANUAL_REVIEW'
    });
  }
});

// 8. Document Upload & OCR Tool
const processDocumentTool = new DynamicStructuredTool({
  name: 'process_document',
  description: 'Extract data from uploaded documents using OCR',
  schema: z.object({
    documentUrl: z.string().describe('S3 URL of uploaded document'),
    documentType: z.enum(['emirates_id', 'passport', 'salary_certificate', 'bank_statement'])
  }),
  func: async ({ documentUrl, documentType }) => {
    const ocrService = new AzureOCRService({
      endpoint: process.env.AZURE_OCR_ENDPOINT,
      apiKey: process.env.AZURE_OCR_KEY
    });
    
    const extractedData = await ocrService.processDocument(documentUrl, documentType);
    
    return JSON.stringify({
      documentType,
      extracted: extractedData.fields,
      confidence: extractedData.confidence,
      verified: extractedData.confidence > 0.9
    });
  }
});

// Export all tools
export const ENBDLoanProcessingTools = [
  getCustomerInfoTool,
  checkCreditBureauTool,
  verifySalaryTool,
  getPropertyValuationTool,
  calculateLoanTool,
  assessLoanRiskTool,
  checkComplianceTool,
  processDocumentTool
];
```

### Using Tools in Agent

```typescript
import { createOpenAIFunctionsAgent } from 'langchain/agents';
import { ChatOpenAI } from '@langchain/openai';
import { ChatPromptTemplate } from '@langchain/core/prompts';

// Create agent with tools
const model = new ChatOpenAI({
  modelName: 'gpt-4',
  temperature: 0
});

const prompt = ChatPromptTemplate.fromMessages([
  ['system', 'You are a loan processing specialist at Emirates NBD. Use the available tools to process loan applications.'],
  ['human', '{input}'],
  ['placeholder', '{agent_scratchpad}']
]);

const agent = await createOpenAIFunctionsAgent({
  llm: model,
  tools: ENBDLoanProcessingTools,
  prompt
});

// Execute
const executor = new AgentExecutor({
  agent,
  tools: ENBDLoanProcessingTools
});

const result = await executor.invoke({
  input: 'Process loan application for customer CUST-12345. They want AED 500,000 for home purchase.'
});

console.log(result.output);
```

---

## Best Practices

### 1. Tool Design

✅ **Do:**
- Make tool descriptions clear and specific
- Use structured schemas (Zod) for type safety
- Return JSON for complex data
- Handle errors gracefully
- Log tool executions
- Add authentication/authorization

❌ **Don't:**
- Create overly complex tools (split into smaller ones)
- Return unstructured text for complex data
- Expose sensitive operations without checks
- Forget to validate inputs

### 2. Tool Naming

```typescript
// ✅ Good - Clear, specific names
'get_customer_credit_score'
'verify_emirates_id'
'calculate_loan_emi'
'send_approval_email'

// ❌ Bad - Vague names
'get_data'
'check_something'
'do_calculation'
'send'
```

### 3. Tool Descriptions

```typescript
// ✅ Good - Detailed, with examples
{
  name: 'verify_emirates_id',
  description: `Verify Emirates ID with ICP (Identity and Citizenship Platform).
  Returns verification status, name match, and expiry date.
  Example: verify_emirates_id("784-1234-5678901-2", "1990-05-15")`
}

// ❌ Bad - Too vague
{
  name: 'verify_id',
  description: 'Verifies ID'
}
```

### 4. Error Handling

```typescript
// ✅ Good - Graceful error handling
func: async ({ customerId }) => {
  try {
    const data = await fetchData(customerId);
    return JSON.stringify({ success: true, data });
  } catch (error) {
    return JSON.stringify({ 
      success: false, 
      error: error.message,
      customerIdAttempted: customerId 
    });
  }
}

// ❌ Bad - Throw errors
func: async ({ customerId }) => {
  const data = await fetchData(customerId);  // Throws on error
  return JSON.stringify(data);
}
```

### 5. Security

```typescript
// ✅ Good - Check permissions
func: async ({ customerId, requesterId }) => {
  // Verify requester has access to this customer
  const hasAccess = await checkPermission(requesterId, customerId);
  
  if (!hasAccess) {
    return JSON.stringify({ 
      error: 'Access denied',
      code: 'PERMISSION_DENIED' 
    });
  }
  
  const data = await fetchSensitiveData(customerId);
  return JSON.stringify(data);
}
```

---

## 8. QA Automation Tools

### Complete QA Tools Suite

```typescript
import { DynamicStructuredTool } from 'langchain/tools';
import { z } from 'zod';
import { chromium } from 'playwright';
import AWS from 'aws-sdk';
import { Pinecone } from '@pinecone-database/pinecone';
import puppeteer from 'puppeteer';

/**
 * Category 1: JIRA Integration Tools
 */

// Tool 1.1: Fetch JIRA Ticket
const fetchJiraTicketTool = new DynamicStructuredTool({
  name: 'fetch_jira_ticket',
  description: 'Fetches JIRA ticket details including summary, description, and acceptance criteria',
  schema: z.object({
    ticketId: z.string().describe('JIRA ticket ID (e.g., QA-1234)'),
  }),
  func: async ({ ticketId }) => {
    try {
      const response = await fetch(
        `https://jira.company.com/rest/api/2/issue/${ticketId}`,
        {
          headers: {
            Authorization: `Bearer ${process.env.JIRA_TOKEN}`,
            'Content-Type': 'application/json',
          },
        }
      );

      if (!response.ok) {
        throw new Error(`JIRA API returned ${response.status}`);
      }

      const data = await response.json();
      
      return JSON.stringify({
        success: true,
        ticketId: data.key,
        summary: data.fields.summary,
        description: data.fields.description,
        acceptanceCriteria: data.fields.customfield_10100 || [],
        status: data.fields.status.name,
        priority: data.fields.priority.name,
        assignee: data.fields.assignee?.displayName || 'Unassigned',
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `Failed to fetch JIRA ticket: ${error.message}`,
      });
    }
  },
});

// Tool 1.2: Update JIRA Ticket
const updateJiraTicketTool = new DynamicStructuredTool({
  name: 'update_jira_ticket',
  description: 'Adds a comment to a JIRA ticket with test results',
  schema: z.object({
    ticketId: z.string(),
    comment: z.string().describe('Comment text to add'),
  }),
  func: async ({ ticketId, comment }) => {
    try {
      const response = await fetch(
        `https://jira.company.com/rest/api/2/issue/${ticketId}/comment`,
        {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${process.env.JIRA_TOKEN}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({ body: comment }),
        }
      );

      if (!response.ok) {
        throw new Error(`JIRA API returned ${response.status}`);
      }

      return JSON.stringify({
        success: true,
        message: 'Comment added to JIRA ticket',
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `Failed to update JIRA: ${error.message}`,
      });
    }
  },
});

/**
 * Category 2: SOP (Standard Operating Procedure) Tools
 */

// Tool 2.1: Search SOP Database (Vector Search)
const searchSopDatabaseTool = new DynamicStructuredTool({
  name: 'search_sop_database',
  description: 'Searches SOP database using semantic search to find relevant testing standards',
  schema: z.object({
    query: z.string().describe('Search query describing the testing requirement'),
    topK: z.number().optional().describe('Number of results to return (default: 3)'),
  }),
  func: async ({ query, topK = 3 }) => {
    try {
      // Generate embedding for query
      const embeddingResponse = await fetch(
        'https://api.openai.com/v1/embeddings',
        {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            model: 'text-embedding-ada-002',
            input: query,
          }),
        }
      );

      const embeddingData = await embeddingResponse.json();
      const embedding = embeddingData.data[0].embedding;

      // Search Pinecone
      const pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY! });
      const index = pinecone.index('sop-embeddings');

      const searchResults = await index.query({
        vector: embedding,
        topK,
        includeMetadata: true,
      });

      const sops = searchResults.matches?.map((match) => ({
        sopId: match.id,
        title: match.metadata?.title,
        content: match.metadata?.content,
        category: match.metadata?.category,
        relevanceScore: match.score,
      }));

      return JSON.stringify({
        success: true,
        query,
        results: sops,
        count: sops?.length || 0,
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `SOP search failed: ${error.message}`,
      });
    }
  },
});

// Tool 2.2: Retrieve SOP Section
const retrieveSopSectionTool = new DynamicStructuredTool({
  name: 'retrieve_sop_section',
  description: 'Retrieves a specific section from a SOP document',
  schema: z.object({
    sopId: z.string().describe('SOP document ID'),
    section: z.string().optional().describe('Specific section to retrieve'),
  }),
  func: async ({ sopId, section }) => {
    try {
      // Fetch from document store
      const response = await fetch(
        `https://api.company.com/sops/${sopId}${section ? `?section=${section}` : ''}`,
        {
          headers: {
            Authorization: `Bearer ${process.env.API_TOKEN}`,
          },
        }
      );

      const data = await response.json();

      return JSON.stringify({
        success: true,
        sopId,
        section: section || 'full_document',
        content: data.content,
        lastUpdated: data.lastUpdated,
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `Failed to retrieve SOP: ${error.message}`,
      });
    }
  },
});

/**
 * Category 3: Playwright Test Execution Tools
 */

// Tool 3.1: Execute Playwright Test
const executePlaywrightTestTool = new DynamicStructuredTool({
  name: 'execute_playwright_test',
  description: 'Executes a Playwright test script and returns results',
  schema: z.object({
    testScript: z.string().describe('Playwright test script to execute'),
    testId: z.string().describe('Unique test identifier'),
    baseUrl: z.string().optional().describe('Base URL for testing'),
  }),
  func: async ({ testScript, testId, baseUrl = 'https://staging.company.com' }) => {
    let browser;
    const screenshots: string[] = [];
    const logs: string[] = [];

    try {
      browser = await chromium.launch({ headless: true });
      const context = await browser.newContext({
        viewport: { width: 1920, height: 1080 },
      });
      const page = await context.newPage();

      // Capture console logs
      page.on('console', (msg) => {
        logs.push(`[Console ${msg.type()}] ${msg.text()}`);
      });

      // Navigate to base URL
      await page.goto(baseUrl);

      // Execute the test script
      const testFunction = new Function('page', 'expect', testScript);
      await testFunction(page, require('@playwright/test').expect);

      // Capture success screenshot
      const screenshotPath = `/tmp/test_${testId}_success.png`;
      await page.screenshot({ path: screenshotPath, fullPage: true });
      screenshots.push(screenshotPath);

      await browser.close();

      return JSON.stringify({
        success: true,
        testId,
        status: 'PASSED',
        screenshots,
        logs,
        executionTime: new Date().toISOString(),
      });
    } catch (error) {
      // Capture failure screenshot
      if (browser) {
        const page = (await browser.contexts())[0]?.pages()[0];
        if (page) {
          const screenshotPath = `/tmp/test_${testId}_failure.png`;
          await page.screenshot({ path: screenshotPath, fullPage: true });
          screenshots.push(screenshotPath);
        }
        await browser.close();
      }

      logs.push(`[ERROR] ${error.message}`);

      return JSON.stringify({
        success: false,
        testId,
        status: 'FAILED',
        error: error.message,
        screenshots,
        logs,
        executionTime: new Date().toISOString(),
      });
    }
  },
});

// Tool 3.2: Capture Screenshot
const captureScreenshotTool = new DynamicStructuredTool({
  name: 'capture_screenshot',
  description: 'Captures a screenshot of a specific page element or full page',
  schema: z.object({
    url: z.string().describe('URL to capture'),
    selector: z.string().optional().describe('CSS selector for specific element'),
    fullPage: z.boolean().optional().describe('Capture full page (default: true)'),
  }),
  func: async ({ url, selector, fullPage = true }) => {
    let browser;
    try {
      browser = await chromium.launch({ headless: true });
      const page = await browser.newPage();
      await page.goto(url, { waitUntil: 'networkidle' });

      const timestamp = Date.now();
      const screenshotPath = `/tmp/screenshot_${timestamp}.png`;

      if (selector) {
        const element = await page.locator(selector);
        await element.screenshot({ path: screenshotPath });
      } else {
        await page.screenshot({ path: screenshotPath, fullPage });
      }

      await browser.close();

      return JSON.stringify({
        success: true,
        screenshotPath,
        url,
        selector: selector || 'full_page',
      });
    } catch (error) {
      if (browser) await browser.close();
      return JSON.stringify({
        success: false,
        error: `Screenshot capture failed: ${error.message}`,
      });
    }
  },
});

/**
 * Category 4: AWS S3 Artifact Management Tools
 */

// Tool 4.1: Upload to S3
const uploadToS3Tool = new DynamicStructuredTool({
  name: 'upload_to_s3',
  description: 'Uploads a file (screenshot, log, report) to AWS S3',
  schema: z.object({
    filePath: z.string().describe('Local file path to upload'),
    s3Key: z.string().describe('S3 object key (path in bucket)'),
    contentType: z.string().optional().describe('MIME type (default: image/png)'),
  }),
  func: async ({ filePath, s3Key, contentType = 'image/png' }) => {
    try {
      const s3 = new AWS.S3({
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
        region: process.env.AWS_REGION,
      });

      const fileContent = await require('fs').promises.readFile(filePath);

      await s3
        .putObject({
          Bucket: process.env.S3_BUCKET!,
          Key: s3Key,
          Body: fileContent,
          ContentType: contentType,
        })
        .promise();

      const s3Url = `https://${process.env.S3_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${s3Key}`;

      return JSON.stringify({
        success: true,
        s3Url,
        s3Key,
        bucket: process.env.S3_BUCKET,
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `S3 upload failed: ${error.message}`,
      });
    }
  },
});

// Tool 4.2: Generate Signed URL
const generateSignedUrlTool = new DynamicStructuredTool({
  name: 'generate_signed_url',
  description: 'Generates a pre-signed URL for secure S3 object access',
  schema: z.object({
    s3Key: z.string().describe('S3 object key'),
    expiresIn: z.number().optional().describe('URL expiration in seconds (default: 604800 = 7 days)'),
  }),
  func: async ({ s3Key, expiresIn = 604800 }) => {
    try {
      const s3 = new AWS.S3({
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
        region: process.env.AWS_REGION,
      });

      const signedUrl = s3.getSignedUrl('getObject', {
        Bucket: process.env.S3_BUCKET!,
        Key: s3Key,
        Expires: expiresIn,
      });

      return JSON.stringify({
        success: true,
        signedUrl,
        expiresIn,
        expiresAt: new Date(Date.now() + expiresIn * 1000).toISOString(),
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `Signed URL generation failed: ${error.message}`,
      });
    }
  },
});

/**
 * Category 5: PDF Report Generation Tools
 */

// Tool 5.1: Create PDF Report
const createPdfReportTool = new DynamicStructuredTool({
  name: 'create_pdf_report',
  description: 'Generates a PDF report from test results with embedded screenshots',
  schema: z.object({
    reportData: z.object({
      ticketId: z.string(),
      summary: z.string(),
      passed: z.number(),
      failed: z.number(),
      coverage: z.number(),
      screenshotUrls: z.array(z.string()),
      logs: z.array(z.string()),
    }),
  }),
  func: async ({ reportData }) => {
    try {
      const browser = await puppeteer.launch({ headless: true });
      const page = await browser.newPage();

      const html = `
<!DOCTYPE html>
<html>
<head>
  <title>QA Test Report - ${reportData.ticketId}</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 40px; }
    h1 { color: #2c3e50; border-bottom: 3px solid #3498db; padding-bottom: 10px; }
    .summary { background: #ecf0f1; padding: 20px; margin: 20px 0; border-radius: 8px; }
    .summary-item { margin: 10px 0; font-size: 16px; }
    .passed { color: #27ae60; font-weight: bold; }
    .failed { color: #e74c3c; font-weight: bold; }
    .screenshot { margin: 20px 0; page-break-inside: avoid; }
    .screenshot img { max-width: 100%; border: 2px solid #bdc3c7; border-radius: 4px; }
    .screenshot-title { font-weight: bold; margin: 10px 0; color: #34495e; }
    .logs { background: #2c3e50; color: #ecf0f1; padding: 15px; border-radius: 4px; font-family: monospace; font-size: 12px; white-space: pre-wrap; }
    .footer { margin-top: 40px; text-align: center; color: #7f8c8d; font-size: 12px; }
  </style>
</head>
<body>
  <h1>QA Test Automation Report</h1>
  
  <div class="summary">
    <div class="summary-item"><strong>Ticket:</strong> ${reportData.ticketId}</div>
    <div class="summary-item"><strong>Summary:</strong> ${reportData.summary}</div>
    <div class="summary-item"><strong>Test Coverage:</strong> ${reportData.coverage}%</div>
    <div class="summary-item">
      <span class="passed">✓ Passed: ${reportData.passed}</span> | 
      <span class="failed">✗ Failed: ${reportData.failed}</span>
    </div>
    <div class="summary-item"><strong>Generated:</strong> ${new Date().toLocaleString()}</div>
  </div>
  
  <h2>Test Screenshots</h2>
  ${reportData.screenshotUrls
    .map(
      (url, idx) => `
    <div class="screenshot">
      <div class="screenshot-title">Screenshot ${idx + 1}</div>
      <img src="${url}" alt="Test screenshot ${idx + 1}" />
    </div>
  `
    )
    .join('')}
  
  <h2>Execution Logs</h2>
  <div class="logs">${reportData.logs.join('\n')}</div>
  
  <div class="footer">
    <p>Generated by QA Automation System | Powered by AI</p>
  </div>
</body>
</html>`;

      await page.setContent(html, { waitUntil: 'networkidle0' });
      const pdfBuffer = await page.pdf({
        format: 'A4',
        printBackground: true,
        margin: { top: '20px', right: '20px', bottom: '20px', left: '20px' },
      });

      await browser.close();

      // Save PDF to temp file
      const pdfPath = `/tmp/report_${reportData.ticketId}_${Date.now()}.pdf`;
      await require('fs').promises.writeFile(pdfPath, pdfBuffer);

      return JSON.stringify({
        success: true,
        pdfPath,
        pages: Math.ceil(reportData.screenshotUrls.length + 2),
        size: pdfBuffer.length,
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `PDF generation failed: ${error.message}`,
      });
    }
  },
});

// Tool 5.2: Send Email Notification
const sendEmailNotificationTool = new DynamicStructuredTool({
  name: 'send_email_notification',
  description: 'Sends email notification with test report to QA team',
  schema: z.object({
    to: z.array(z.string()).describe('Recipient email addresses'),
    subject: z.string(),
    body: z.string().describe('HTML email body'),
    attachmentUrl: z.string().optional().describe('PDF report URL'),
  }),
  func: async ({ to, subject, body, attachmentUrl }) => {
    try {
      const sgMail = require('@sendgrid/mail');
      sgMail.setApiKey(process.env.SENDGRID_API_KEY);

      const msg = {
        to,
        from: 'automation@company.com',
        subject,
        html: body,
      };

      await sgMail.send(msg);

      return JSON.stringify({
        success: true,
        recipients: to,
        subject,
        sentAt: new Date().toISOString(),
      });
    } catch (error) {
      return JSON.stringify({
        success: false,
        error: `Email notification failed: ${error.message}`,
      });
    }
  },
});

/**
 * Tool Usage Example: Complete QA Flow
 */
async function qaAutomationExample() {
  // 1. Fetch JIRA ticket
  const ticketResult = await fetchJiraTicketTool.func({ ticketId: 'QA-1234' });
  const ticket = JSON.parse(ticketResult);

  // 2. Search for relevant SOPs
  const sopResult = await searchSopDatabaseTool.func({
    query: ticket.description,
    topK: 3,
  });
  const sops = JSON.parse(sopResult);

  // 3. Execute Playwright test
  const testScript = `
    await page.goto('https://staging.company.com/login');
    await page.fill('#email', 'test@example.com');
    await page.fill('#password', 'password123');
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL(/.*dashboard/);
  `;

  const testResult = await executePlaywrightTestTool.func({
    testScript,
    testId: 'TC-001',
  });
  const test = JSON.parse(testResult);

  // 4. Upload screenshot to S3
  const uploadResult = await uploadToS3Tool.func({
    filePath: test.screenshots[0],
    s3Key: `qa-automation/QA-1234/screenshot_001.png`,
    contentType: 'image/png',
  });
  const upload = JSON.parse(uploadResult);

  // 5. Generate signed URL
  const urlResult = await generateSignedUrlTool.func({
    s3Key: `qa-automation/QA-1234/screenshot_001.png`,
  });
  const signedUrl = JSON.parse(urlResult);

  // 6. Create PDF report
  const pdfResult = await createPdfReportTool.func({
    reportData: {
      ticketId: 'QA-1234',
      summary: ticket.summary,
      passed: 7,
      failed: 1,
      coverage: 85,
      screenshotUrls: [signedUrl.signedUrl],
      logs: test.logs,
    },
  });
  const pdf = JSON.parse(pdfResult);

  // 7. Upload PDF to S3
  await uploadToS3Tool.func({
    filePath: pdf.pdfPath,
    s3Key: `qa-automation/QA-1234/report.pdf`,
    contentType: 'application/pdf',
  });

  // 8. Send email notification
  await sendEmailNotificationTool.func({
    to: ['qa-team@company.com'],
    subject: `QA Test Report - ${ticket.ticketId}`,
    body: `
      <h2>Test Execution Complete</h2>
      <p><strong>Ticket:</strong> ${ticket.ticketId}</p>
      <p><strong>Passed:</strong> 7 | <strong>Failed:</strong> 1</p>
      <p><a href="${signedUrl.signedUrl}">View Full Report</a></p>
    `,
  });

  // 9. Update JIRA ticket
  await updateJiraTicketTool.func({
    ticketId: 'QA-1234',
    comment: `Automated testing completed. Passed: 7, Failed: 1. Report: ${upload.s3Url}`,
  });

  console.log('✅ Complete QA automation flow executed successfully');
}
```

### QA Tools Summary

| Tool | Category | External System | Purpose |
|------|----------|----------------|---------|
| `fetch_jira_ticket` | JIRA | JIRA REST API | Fetch ticket details |
| `update_jira_ticket` | JIRA | JIRA REST API | Add comments |
| `search_sop_database` | SOP | Pinecone | Vector search SOPs |
| `retrieve_sop_section` | SOP | Document Store | Get SOP content |
| `execute_playwright_test` | Testing | Playwright | Run browser tests |
| `capture_screenshot` | Testing | Playwright | Screenshot capture |
| `upload_to_s3` | Artifacts | AWS S3 | Upload files |
| `generate_signed_url` | Artifacts | AWS S3 | Secure URLs |
| `create_pdf_report` | Reporting | Puppeteer | Generate PDFs |
| `send_email_notification` | Notification | SendGrid | Email alerts |

---

## 9. Summary

### Key Concepts

1. **Tools extend agent capabilities** beyond text generation
2. **Each agent can have specialized tools** for their role
3. **Tools should be simple, focused, and well-documented**
4. **Use schemas for type safety and validation**
5. **Handle errors gracefully** and return structured data

### Tool Categories

| Category | Examples | Use Case |
|----------|----------|----------|
| Information | Database queries, API calls | Fetch data |
| Action | Send email, create ticket | Perform operations |
| Computation | Calculate EMI, DTI | Complex calculations |
| Verification | Check ID, sanctions | Validate information |

### Banking Tools (ENBD)

- ✅ Customer lookup (CRM)
- ✅ Credit bureau integration
- ✅ Salary verification (WPS)
- ✅ Property valuation (DLD)
- ✅ Loan calculators
- ✅ Risk assessment
- ✅ Compliance checks (AML/KYC)
- ✅ Document OCR

### QA Automation Tools

- ✅ JIRA integration (fetch, update tickets)
- ✅ SOP search (Pinecone vector DB)
- ✅ Playwright test execution
- ✅ Screenshot capture
- ✅ AWS S3 artifact management
- ✅ PDF report generation (Puppeteer)
- ✅ Email notifications (SendGrid)

---

**Next Steps:**
- See `QA_Automation_Scenario.md` for complete 6-agent implementation
- Check `Q51_Multi_Agent_Systems_Architecture.md` for tools in multi-agent systems
- Review `Architecture_Diagrams.md` for tool flow visualization
- Practice creating custom tools for your use case

---

**Document Version**: 2.0  
**Last Updated**: November 2025  
**Scenarios**: QA Automation + ENBD Banking  
**For**: GenAI Interview Preparation

---

*End of Tools in Agent Systems Document*
