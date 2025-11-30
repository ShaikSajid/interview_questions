# LangGraph Basics - Complete Guide

## What is LangGraph?

LangGraph is a library for building stateful, multi-actor applications with LLMs. It extends LangChain with the ability to create cyclical graphs, which are essential for most agentic workflows.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [State Management](#state-management)
3. [Building Your First Graph](#building-your-first-graph)
4. [Nodes and Edges](#nodes-and-edges)
5. [Conditional Routing](#conditional-routing)
6. [Persistence and Checkpoints](#persistence-and-checkpoints)
7. [ENBD Banking Examples](#enbd-banking-examples)
8. [Advanced Patterns](#advanced-patterns)

---

## Core Concepts

### 1. StateGraph

The fundamental building block of LangGraph. It manages state transitions through your workflow.

```typescript
import { StateGraph, END } from '@langchain/langgraph';

// Define your state structure
interface MyState {
  messages: string[];
  currentStep: string;
  data: any;
}

// Create a graph
const workflow = new StateGraph<MyState>({
  channels: {
    messages: null,
    currentStep: null,
    data: null
  }
});
```

### 2. State

State is the shared data that flows through your graph. Each node can read from and write to the state.

```
Initial State → Node A → Updated State → Node B → Final State
```

### 3. Nodes

Nodes are functions that process the state and return updates.

```typescript
async function myNode(state: MyState): Promise<Partial<MyState>> {
  // Process state
  const result = await someOperation(state.data);
  
  // Return state updates
  return {
    messages: [...state.messages, 'Completed operation'],
    data: result
  };
}
```

### 4. Edges

Edges define the flow between nodes:
- **Normal edges**: Direct connections (`A → B`)
- **Conditional edges**: Dynamic routing based on state
- **Entry point**: Where the graph starts
- **END**: Terminal node

---

## State Management

### State Schema

```typescript
interface LoanApplicationState {
  // Application data
  applicationId: string;
  customerName: string;
  requestedAmount: number;
  
  // Processing results
  creditScore?: number;
  riskLevel?: string;
  decision?: string;
  
  // Metadata
  currentNode: string;
  timestamp: Date;
  errors: string[];
}
```

### State Updates

State updates are **merged**, not replaced:

```typescript
// Initial state
{
  applicationId: "LOAN-001",
  currentNode: "start"
}

// Node returns
{
  creditScore: 720,
  currentNode: "credit_check"
}

// Resulting state (merged)
{
  applicationId: "LOAN-001",
  creditScore: 720,
  currentNode: "credit_check"
}
```

---

## Building Your First Graph

### Step-by-Step Example: Simple Approval Workflow

```typescript
import { StateGraph, END } from '@langchain/langgraph';

// 1. Define State
interface ApprovalState {
  requestId: string;
  amount: number;
  approved?: boolean;
  reason?: string;
}

// 2. Create Graph
const workflow = new StateGraph<ApprovalState>({
  channels: {
    requestId: null,
    amount: null,
    approved: null,
    reason: null
  }
});

// 3. Define Nodes
async function checkAmount(state: ApprovalState): Promise<Partial<ApprovalState>> {
  console.log(`Checking amount: AED ${state.amount}`);
  
  if (state.amount <= 10000) {
    return {
      approved: true,
      reason: 'Auto-approved (amount <= 10k)'
    };
  } else {
    return {
      approved: false,
      reason: 'Requires manual review (amount > 10k)'
    };
  }
}

// 4. Add Nodes
workflow.addNode('check_amount', checkAmount);

// 5. Define Edges
workflow.addEdge('check_amount', END);

// 6. Set Entry Point
workflow.setEntryPoint('check_amount');

// 7. Compile and Run
const app = workflow.compile();

// Execute
const result = await app.invoke({
  requestId: 'REQ-001',
  amount: 5000
});

console.log(result);
// {
//   requestId: 'REQ-001',
//   amount: 5000,
//   approved: true,
//   reason: 'Auto-approved (amount <= 10k)'
// }
```

---

## Nodes and Edges

### Normal Edges

Direct connections between nodes:

```typescript
workflow.addNode('step1', node1Function);
workflow.addNode('step2', node2Function);
workflow.addNode('step3', node3Function);

// Create linear flow
workflow.addEdge('step1', 'step2');
workflow.addEdge('step2', 'step3');
workflow.addEdge('step3', END);

workflow.setEntryPoint('step1');
```

**Flow**: `step1 → step2 → step3 → END`

### Conditional Edges

Dynamic routing based on state:

```typescript
// Routing function
function router(state: MyState): string {
  if (state.amount > 50000) {
    return 'high_value_review';
  } else if (state.amount > 10000) {
    return 'standard_review';
  } else {
    return 'auto_approve';
  }
}

// Add conditional edge
workflow.addConditionalEdges(
  'check_amount',  // From node
  router,          // Routing function
  {
    'high_value_review': 'high_value_review',
    'standard_review': 'standard_review',
    'auto_approve': 'auto_approve'
  }
);
```

**Flow**: 
```
check_amount → router → high_value_review (if >50k)
                     → standard_review   (if 10k-50k)
                     → auto_approve      (if <10k)
```

---

## Conditional Routing

### Example: Credit Decision Workflow

```typescript
import { StateGraph, END } from '@langchain/langgraph';
import { ChatOpenAI } from '@langchain/openai';

interface CreditState {
  applicationId: string;
  creditScore: number;
  income: number;
  decision?: string;
  route?: string;
}

// Node 1: Get Credit Score
async function getCreditScore(state: CreditState): Promise<Partial<CreditState>> {
  console.log('Fetching credit score...');
  
  // Simulate credit score fetch
  const score = Math.floor(Math.random() * (850 - 300) + 300);
  
  return {
    creditScore: score
  };
}

// Node 2: Auto Approve (High Score)
async function autoApprove(state: CreditState): Promise<Partial<CreditState>> {
  console.log('Auto-approving application...');
  
  return {
    decision: 'APPROVED',
    route: 'auto'
  };
}

// Node 3: Manual Review (Medium Score)
async function manualReview(state: CreditState): Promise<Partial<CreditState>> {
  console.log('Sending for manual review...');
  
  return {
    decision: 'PENDING_REVIEW',
    route: 'manual'
  };
}

// Node 4: Auto Reject (Low Score)
async function autoReject(state: CreditState): Promise<Partial<CreditState>> {
  console.log('Auto-rejecting application...');
  
  return {
    decision: 'REJECTED',
    route: 'auto'
  };
}

// Router: Decide path based on credit score
function creditRouter(state: CreditState): string {
  const { creditScore } = state;
  
  if (creditScore >= 750) {
    return 'auto_approve';
  } else if (creditScore >= 650) {
    return 'manual_review';
  } else {
    return 'auto_reject';
  }
}

// Build the graph
const creditWorkflow = new StateGraph<CreditState>({
  channels: {
    applicationId: null,
    creditScore: null,
    income: null,
    decision: null,
    route: null
  }
});

// Add nodes
creditWorkflow.addNode('get_credit_score', getCreditScore);
creditWorkflow.addNode('auto_approve', autoApprove);
creditWorkflow.addNode('manual_review', manualReview);
creditWorkflow.addNode('auto_reject', autoReject);

// Add conditional routing
creditWorkflow.addConditionalEdges(
  'get_credit_score',
  creditRouter,
  {
    'auto_approve': 'auto_approve',
    'manual_review': 'manual_review',
    'auto_reject': 'auto_reject'
  }
);

// All paths end after decision
creditWorkflow.addEdge('auto_approve', END);
creditWorkflow.addEdge('manual_review', END);
creditWorkflow.addEdge('auto_reject', END);

// Set entry point
creditWorkflow.setEntryPoint('get_credit_score');

// Compile
const creditApp = creditWorkflow.compile();

// Usage
async function processCreditApplication() {
  const result = await creditApp.invoke({
    applicationId: 'APP-2025-001',
    creditScore: 0,  // Will be fetched
    income: 35000
  });
  
  console.log('\n=== Credit Decision ===');
  console.log(`Application ID: ${result.applicationId}`);
  console.log(`Credit Score: ${result.creditScore}`);
  console.log(`Decision: ${result.decision}`);
  console.log(`Route: ${result.route}`);
}

// Run
processCreditApplication();
```

### Visual Flow:

```
                    ┌─────────────────┐
                    │  Entry Point    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ get_credit_score│
                    └────────┬────────┘
                             │
                             ▼
                      [creditRouter]
                             │
            ┌────────────────┼────────────────┐
            │                │                │
    ┌───────▼────────┐  ┌───▼────────┐  ┌───▼────────┐
    │ auto_approve   │  │manual_review│  │auto_reject │
    │ (score>=750)   │  │(650-749)    │  │ (<650)     │
    └───────┬────────┘  └───┬────────┘  └───┬────────┘
            │               │                │
            └───────────────┼────────────────┘
                            │
                            ▼
                         [END]
```

---

## Persistence and Checkpoints

### Checkpointing

LangGraph can save state at each step, enabling:
- Resume from interruption
- Time travel through execution
- Human-in-the-loop workflows

```typescript
import { MemorySaver } from '@langchain/langgraph';

// Create checkpointer
const checkpointer = new MemorySaver();

// Compile with checkpointer
const app = workflow.compile({
  checkpointer
});

// Run with thread ID
const result = await app.invoke(
  { applicationId: 'LOAN-001', amount: 50000 },
  { configurable: { thread_id: 'thread-1' } }
);

// Resume later with same thread_id
const continueResult = await app.invoke(
  null,  // No new input needed
  { configurable: { thread_id: 'thread-1' } }
);
```

### State History

```typescript
// Get all checkpoints for a thread
const history = await app.getStateHistory({
  configurable: { thread_id: 'thread-1' }
});

// Iterate through history
for await (const checkpoint of history) {
  console.log('Checkpoint:', checkpoint);
  console.log('Values:', checkpoint.values);
  console.log('Next:', checkpoint.next);
}
```

---

## ENBD Banking Examples

### Example 1: Loan Approval Workflow

```typescript
import { StateGraph, END } from '@langchain/langgraph';

interface LoanState {
  applicationId: string;
  customerName: string;
  requestedAmount: number;
  monthlyIncome: number;
  creditScore?: number;
  debtToIncome?: number;
  riskLevel?: string;
  decision?: string;
  approvedAmount?: number;
  currentStep: string;
}

// Step 1: Credit Check
async function creditCheck(state: LoanState): Promise<Partial<LoanState>> {
  console.log(`\n[Credit Check] Processing ${state.customerName}...`);
  
  // Simulate credit bureau API call
  const creditScore = Math.floor(Math.random() * (850 - 600) + 600);
  
  console.log(`Credit Score: ${creditScore}`);
  
  return {
    creditScore,
    currentStep: 'credit_check_complete'
  };
}

// Step 2: Calculate DTI (Debt-to-Income)
async function calculateDTI(state: LoanState): Promise<Partial<LoanState>> {
  console.log(`\n[DTI Calculation] Analyzing financials...`);
  
  // Simulate existing debt lookup
  const existingDebt = Math.floor(Math.random() * 10000);
  const monthlyLoanPayment = (state.requestedAmount * 0.05) / 12;
  const totalMonthlyDebt = existingDebt + monthlyLoanPayment;
  const dti = (totalMonthlyDebt / state.monthlyIncome) * 100;
  
  console.log(`DTI Ratio: ${dti.toFixed(2)}%`);
  
  return {
    debtToIncome: dti,
    currentStep: 'dti_calculated'
  };
}

// Step 3: Risk Assessment
async function riskAssessment(state: LoanState): Promise<Partial<LoanState>> {
  console.log(`\n[Risk Assessment] Evaluating risk...`);
  
  const { creditScore = 650, debtToIncome = 30 } = state;
  
  let riskLevel: string;
  
  if (creditScore >= 750 && debtToIncome < 30) {
    riskLevel = 'LOW';
  } else if (creditScore >= 680 && debtToIncome < 40) {
    riskLevel = 'MEDIUM';
  } else {
    riskLevel = 'HIGH';
  }
  
  console.log(`Risk Level: ${riskLevel}`);
  
  return {
    riskLevel,
    currentStep: 'risk_assessed'
  };
}

// Step 4: Final Decision
async function finalDecision(state: LoanState): Promise<Partial<LoanState>> {
  console.log(`\n[Final Decision] Making approval decision...`);
  
  const { creditScore = 650, riskLevel = 'MEDIUM', requestedAmount } = state;
  
  let decision: string;
  let approvedAmount: number;
  
  if (creditScore >= 750 && riskLevel === 'LOW') {
    decision = 'APPROVED';
    approvedAmount = requestedAmount;
  } else if (creditScore >= 680 && riskLevel !== 'HIGH') {
    decision = 'APPROVED';
    approvedAmount = Math.floor(requestedAmount * 0.75); // 75%
  } else if (creditScore >= 650 && riskLevel === 'LOW') {
    decision = 'APPROVED';
    approvedAmount = Math.floor(requestedAmount * 0.5); // 50%
  } else {
    decision = 'REJECTED';
    approvedAmount = 0;
  }
  
  console.log(`Decision: ${decision}`);
  console.log(`Approved Amount: AED ${approvedAmount.toLocaleString()}`);
  
  return {
    decision,
    approvedAmount,
    currentStep: 'complete'
  };
}

// Build the graph
const loanWorkflow = new StateGraph<LoanState>({
  channels: {
    applicationId: null,
    customerName: null,
    requestedAmount: null,
    monthlyIncome: null,
    creditScore: null,
    debtToIncome: null,
    riskLevel: null,
    decision: null,
    approvedAmount: null,
    currentStep: null
  }
});

// Add nodes
loanWorkflow.addNode('credit_check', creditCheck);
loanWorkflow.addNode('calculate_dti', calculateDTI);
loanWorkflow.addNode('risk_assessment', riskAssessment);
loanWorkflow.addNode('final_decision', finalDecision);

// Define flow
loanWorkflow.addEdge('credit_check', 'calculate_dti');
loanWorkflow.addEdge('calculate_dti', 'risk_assessment');
loanWorkflow.addEdge('risk_assessment', 'final_decision');
loanWorkflow.addEdge('final_decision', END);

loanWorkflow.setEntryPoint('credit_check');

// Compile
const loanApp = loanWorkflow.compile();

// Usage
async function processLoanApplication() {
  console.log('='.repeat(60));
  console.log('EMIRATES NBD - LOAN APPLICATION PROCESSING');
  console.log('='.repeat(60));
  
  const result = await loanApp.invoke({
    applicationId: 'LOAN-2025-12345',
    customerName: 'Ahmed Al Maktoum',
    requestedAmount: 250000,
    monthlyIncome: 25000,
    currentStep: 'initiated'
  });
  
  console.log('\n' + '='.repeat(60));
  console.log('FINAL RESULT');
  console.log('='.repeat(60));
  console.log(`Application ID: ${result.applicationId}`);
  console.log(`Customer: ${result.customerName}`);
  console.log(`Requested: AED ${result.requestedAmount.toLocaleString()}`);
  console.log(`Credit Score: ${result.creditScore}`);
  console.log(`DTI Ratio: ${result.debtToIncome?.toFixed(2)}%`);
  console.log(`Risk Level: ${result.riskLevel}`);
  console.log(`\n>>> DECISION: ${result.decision} <<<`);
  console.log(`>>> APPROVED AMOUNT: AED ${result.approvedAmount?.toLocaleString()} <<<`);
  console.log('='.repeat(60));
}

// Run
processLoanApplication();
```

### Example 2: Customer Onboarding with Conditional Routing

```typescript
interface OnboardingState {
  customerId: string;
  customerType: 'retail' | 'corporate' | 'private_banking';
  documentsComplete?: boolean;
  kycComplete?: boolean;
  accountType?: string;
  nextStep?: string;
}

// Node: Document Verification
async function verifyDocuments(state: OnboardingState): Promise<Partial<OnboardingState>> {
  console.log(`Verifying documents for ${state.customerType} customer...`);
  
  // Simulate verification
  const complete = Math.random() > 0.1; // 90% success rate
  
  return {
    documentsComplete: complete
  };
}

// Node: KYC Process (Retail)
async function kycRetail(state: OnboardingState): Promise<Partial<OnboardingState>> {
  console.log('Performing retail KYC...');
  
  return {
    kycComplete: true,
    accountType: 'Retail Savings',
    nextStep: 'complete'
  };
}

// Node: KYC Process (Corporate)
async function kycCorporate(state: OnboardingState): Promise<Partial<OnboardingState>> {
  console.log('Performing corporate KYC (enhanced due diligence)...');
  
  return {
    kycComplete: true,
    accountType: 'Corporate Current',
    nextStep: 'complete'
  };
}

// Node: KYC Process (Private Banking)
async function kycPrivateBanking(state: OnboardingState): Promise<Partial<OnboardingState>> {
  console.log('Performing private banking KYC (highest tier)...');
  
  return {
    kycComplete: true,
    accountType: 'Private Banking Premium',
    nextStep: 'complete'
  };
}

// Node: Request Documents
async function requestDocuments(state: OnboardingState): Promise<Partial<OnboardingState>> {
  console.log('Requesting additional documents...');
  
  return {
    nextStep: 'pending_documents'
  };
}

// Router: Route based on customer type
function customerTypeRouter(state: OnboardingState): string {
  if (!state.documentsComplete) {
    return 'request_documents';
  }
  
  switch (state.customerType) {
    case 'retail':
      return 'kyc_retail';
    case 'corporate':
      return 'kyc_corporate';
    case 'private_banking':
      return 'kyc_private_banking';
    default:
      return 'kyc_retail';
  }
}

// Build graph
const onboardingWorkflow = new StateGraph<OnboardingState>({
  channels: {
    customerId: null,
    customerType: null,
    documentsComplete: null,
    kycComplete: null,
    accountType: null,
    nextStep: null
  }
});

onboardingWorkflow.addNode('verify_documents', verifyDocuments);
onboardingWorkflow.addNode('kyc_retail', kycRetail);
onboardingWorkflow.addNode('kyc_corporate', kycCorporate);
onboardingWorkflow.addNode('kyc_private_banking', kycPrivateBanking);
onboardingWorkflow.addNode('request_documents', requestDocuments);

// Conditional routing after document verification
onboardingWorkflow.addConditionalEdges(
  'verify_documents',
  customerTypeRouter,
  {
    'kyc_retail': 'kyc_retail',
    'kyc_corporate': 'kyc_corporate',
    'kyc_private_banking': 'kyc_private_banking',
    'request_documents': 'request_documents'
  }
);

// End after KYC
onboardingWorkflow.addEdge('kyc_retail', END);
onboardingWorkflow.addEdge('kyc_corporate', END);
onboardingWorkflow.addEdge('kyc_private_banking', END);
onboardingWorkflow.addEdge('request_documents', END);

onboardingWorkflow.setEntryPoint('verify_documents');

const onboardingApp = onboardingWorkflow.compile();

// Usage
async function onboardCustomer(customerType: 'retail' | 'corporate' | 'private_banking') {
  const result = await onboardingApp.invoke({
    customerId: `CUST-${Date.now()}`,
    customerType
  });
  
  console.log('\n=== Onboarding Result ===');
  console.log(`Customer Type: ${result.customerType}`);
  console.log(`Documents Complete: ${result.documentsComplete}`);
  console.log(`KYC Complete: ${result.kycComplete}`);
  console.log(`Account Type: ${result.accountType}`);
  console.log(`Status: ${result.nextStep}`);
}

// Test different customer types
onboardCustomer('retail');
onboardCustomer('corporate');
onboardCustomer('private_banking');
```

---

## Advanced Patterns

### 1. Loops and Cycles

```typescript
// Router with cycle
function shouldContinue(state: MyState): string {
  if (state.attempts < 3 && !state.success) {
    return 'retry';  // Go back to retry node
  } else {
    return 'finish';
  }
}

workflow.addConditionalEdges(
  'process',
  shouldContinue,
  {
    'retry': 'process',  // Loop back
    'finish': 'complete'
  }
);
```

### 2. Human-in-the-Loop

```typescript
async function requiresApproval(state: LoanState): Promise<Partial<LoanState>> {
  if (state.requestedAmount > 100000) {
    // Pause and wait for human approval
    return {
      nextStep: 'pending_approval',
      humanReview: true
    };
  }
  
  return {
    nextStep: 'auto_approved',
    humanReview: false
  };
}

// Later, resume after human approves
const result = await app.invoke(
  { approved: true },  // Human decision
  { configurable: { thread_id: 'loan-123' } }
);
```

### 3. Parallel Branches (Fan-out/Fan-in)

```typescript
// Not directly supported, but can be simulated
async function parallelNode(state: MyState): Promise<Partial<MyState>> {
  // Execute multiple operations in parallel
  const [result1, result2, result3] = await Promise.all([
    operation1(state),
    operation2(state),
    operation3(state)
  ]);
  
  return {
    result1,
    result2,
    result3
  };
}
```

---

## Key Takeaways

### ✅ When to Use LangGraph

- Complex workflows with conditional logic
- Need to maintain state across multiple steps
- Require checkpointing/resume capability
- Human-in-the-loop workflows
- Cyclical/loop-based processes

### ❌ When NOT to Use LangGraph

- Simple linear chains (use regular LangChain)
- One-off LLM calls
- No state management needed
- Performance-critical single operations

### Best Practices

1. **Keep nodes focused**: Each node should do one thing well
2. **Use type safety**: Define clear TypeScript interfaces for state
3. **Log extensively**: Track state changes for debugging
4. **Handle errors**: Add try-catch in nodes
5. **Test nodes individually**: Unit test each node function
6. **Use conditional routing**: Leverage dynamic paths
7. **Checkpoint important states**: Enable recovery

---

## Quick Reference

### Basic Structure

```typescript
// 1. Define state
interface State { ... }

// 2. Create graph
const workflow = new StateGraph<State>({ channels: {...} });

// 3. Add nodes
workflow.addNode('node_name', nodeFunction);

// 4. Add edges
workflow.addEdge('node1', 'node2');

// 5. Set entry
workflow.setEntryPoint('node1');

// 6. Compile
const app = workflow.compile();

// 7. Run
const result = await app.invoke(initialState);
```

### Common Patterns

```typescript
// Normal edge
workflow.addEdge('a', 'b');

// Conditional edge
workflow.addConditionalEdges('a', routerFn, { 'path1': 'b', 'path2': 'c' });

// End
workflow.addEdge('final', END);

// Entry
workflow.setEntryPoint('start');
```

---

## 8. Example 3: QA Automation Workflow

### Complete QA Test Automation with LangGraph

```typescript
import { StateGraph, END } from '@langchain/langgraph';
import { ChatOpenAI } from '@langchain/openai';
import { DynamicStructuredTool } from 'langchain/tools';
import { z } from 'zod';

/**
 * QA Automation State Interface
 */
interface QAAutomationState {
  ticketId: string;
  ticketDetails?: {
    summary: string;
    description: string;
    acceptanceCriteria: string[];
  };
  testPlan?: {
    testCases: Array<{
      id: string;
      title: string;
      priority: 'high' | 'medium' | 'low';
    }>;
    coverage: number;
  };
  testResults?: {
    passed: number;
    failed: number;
    screenshots: string[];
  };
  pdfReportUrl?: string;
  manualReview?: boolean;
  retryCount?: number;
  error?: string;
}

/**
 * Node 1: JIRA Reader
 */
async function jiraReaderNode(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
  console.log(`[JIRA Reader] Fetching ticket ${state.ticketId}...`);

  const model = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });

  // Simulate JIRA API call
  const ticketDetails = {
    summary: 'Implement user login functionality',
    description: 'Add login form with email and password fields. Include validation and error handling.',
    acceptanceCriteria: [
      'User can enter email and password',
      'Form validates input before submission',
      'Success redirects to dashboard',
      'Errors show proper messages',
      'Remember me checkbox works',
    ],
  };

  return {
    ticketDetails,
  };
}

/**
 * Node 2: Test Generator
 */
async function testGeneratorNode(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
  console.log('[Test Generator] Creating test plan...');

  const model = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });

  const prompt = `Based on the following requirements, create a test plan:

Summary: ${state.ticketDetails?.summary}
Description: ${state.ticketDetails?.description}
Acceptance Criteria:
${state.ticketDetails?.acceptanceCriteria.join('\n')}

Generate 5-8 test cases with:
1. Test ID
2. Title
3. Priority (high/medium/low)

Format as JSON array.`;

  const response = await model.invoke(prompt);
  const testCases = JSON.parse(response.content as string);

  const coverage = (testCases.length / (state.ticketDetails?.acceptanceCriteria.length || 1)) * 100;

  return {
    testPlan: {
      testCases,
      coverage: Math.min(coverage, 100),
    },
  };
}

/**
 * Node 3: Manual Review (Human-in-the-Loop)
 */
async function manualReviewNode(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
  console.log('[Manual Review] Test plan requires human approval...');
  console.log(`Test cases: ${state.testPlan?.testCases.length}`);
  console.log(`Coverage: ${state.testPlan?.coverage}%`);

  // In production, this would trigger a UI for human approval
  // For demo, we'll simulate approval
  console.log('✓ Manual review approved (simulated)');

  return {
    manualReview: true,
  };
}

/**
 * Node 4: Test Executor
 */
async function testExecutorNode(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
  console.log('[Test Executor] Running Playwright tests...');

  // Simulate test execution
  const totalTests = state.testPlan?.testCases.length || 0;
  const passed = Math.floor(totalTests * 0.875); // 87.5% pass rate
  const failed = totalTests - passed;

  const screenshots = Array.from({ length: totalTests }, (_, i) => `/tmp/screenshot_${i + 1}.png`);

  return {
    testResults: {
      passed,
      failed,
      screenshots,
    },
  };
}

/**
 * Node 5: Report Generator
 */
async function reportGeneratorNode(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
  console.log('[Report Generator] Creating PDF report...');

  // Simulate PDF generation and S3 upload
  const pdfReportUrl = `https://qa-bucket.s3.amazonaws.com/reports/${state.ticketId}_${Date.now()}.pdf`;

  // Simulate JIRA update
  console.log(`✓ Updated JIRA ticket ${state.ticketId} with results`);

  // Simulate email notification
  console.log('✓ Email notification sent to QA team');

  return {
    pdfReportUrl,
  };
}

/**
 * Conditional Routing Functions
 */

// Route 1: Should we require manual review?
function shouldManualReview(state: QAAutomationState): string {
  const complexity = state.testPlan?.testCases.length || 0;

  if (complexity > 10) {
    console.log(`[Router] Complexity ${complexity} > 10, routing to manual review`);
    return 'manualReview';
  }

  console.log(`[Router] Complexity ${complexity} <= 10, proceeding to execution`);
  return 'testExecutor';
}

// Route 2: Should we retry on high failure rate?
function shouldRetry(state: QAAutomationState): string {
  const passed = state.testResults?.passed || 0;
  const failed = state.testResults?.failed || 0;
  const total = passed + failed;
  const failureRate = total > 0 ? failed / total : 0;

  const retryCount = state.retryCount || 0;

  if (failureRate > 0.5 && retryCount < 3) {
    console.log(`[Router] Failure rate ${(failureRate * 100).toFixed(1)}% > 50%, retry ${retryCount + 1}/3`);
    return 'testGenerator'; // Regenerate tests
  }

  console.log(`[Router] Acceptable failure rate or max retries reached, completing workflow`);
  return END;
}

/**
 * Build the Complete Workflow
 */
function createQaAutomationWorkflow() {
  // Define state structure
  const workflow = new StateGraph<QAAutomationState>({
    channels: {
      ticketId: null,
      ticketDetails: null,
      testPlan: null,
      testResults: null,
      pdfReportUrl: null,
      manualReview: null,
      retryCount: null,
      error: null,
    },
  });

  // Add nodes
  workflow.addNode('jiraReader', jiraReaderNode);
  workflow.addNode('testGenerator', testGeneratorNode);
  workflow.addNode('manualReview', manualReviewNode);
  workflow.addNode('testExecutor', testExecutorNode);
  workflow.addNode('reportGenerator', reportGeneratorNode);

  // Define edges
  workflow.setEntryPoint('jiraReader');
  workflow.addEdge('jiraReader', 'testGenerator');

  // Conditional: Manual review for complex tickets
  workflow.addConditionalEdges('testGenerator', shouldManualReview, {
    manualReview: 'manualReview',
    testExecutor: 'testExecutor',
  });

  workflow.addEdge('manualReview', 'testExecutor');
  workflow.addEdge('testExecutor', 'reportGenerator');

  // Conditional: Retry on high failure rate
  workflow.addConditionalEdges('reportGenerator', shouldRetry, {
    testGenerator: 'testGenerator', // Retry
    [END]: END, // Complete
  });

  return workflow.compile();
}

/**
 * Main execution
 */
async function main() {
  console.log('=== QA Automation Workflow with LangGraph ===\n');

  const app = createQaAutomationWorkflow();

  const result = await app.invoke({
    ticketId: 'QA-1234',
    retryCount: 0,
  });

  console.log('\n=== Final Result ===');
  console.log(`Ticket: ${result.ticketId}`);
  console.log(`Summary: ${result.ticketDetails?.summary}`);
  console.log(`Test Cases: ${result.testPlan?.testCases.length}`);
  console.log(`Coverage: ${result.testPlan?.coverage}%`);
  console.log(`Passed: ${result.testResults?.passed}`);
  console.log(`Failed: ${result.testResults?.failed}`);
  console.log(`PDF Report: ${result.pdfReportUrl}`);
  console.log(`Manual Review: ${result.manualReview ? 'Yes' : 'No'}`);
}

// main().catch(console.error);
```

### QA Workflow Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│              QA AUTOMATION WORKFLOW                             │
└─────────────────────────────────────────────────────────────────┘

START
  │
  ▼
┌──────────────┐
│ JIRA Reader  │  Fetches ticket details
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Test Gen     │  Creates test plan with coverage
└──────┬───────┘
       │
       ▼
   ┌────────┐
   │ Route? │ ◄─── Conditional 1: complexity > 10?
   └───┬────┘
       │ Yes     │ No
       ▼         │
   ┌─────────┐  │
   │ Manual  │  │
   │ Review  │  │
   └────┬────┘  │
        │       │
        └───────┘
            │
            ▼
    ┌──────────────┐
    │ Test Exec    │  Runs Playwright tests
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ Report Gen   │  Creates PDF, emails team
    └──────┬───────┘
           │
           ▼
       ┌────────┐
       │ Route? │ ◄─── Conditional 2: failure > 50% & retries < 3?
       └───┬────┘
           │ Yes (retry)    │ No (done)
           ▼                ▼
      Back to Test Gen     END
```

### State Evolution Example

```typescript
// Initial State
{
  ticketId: "QA-1234",
  retryCount: 0
}

// After JIRA Reader
{
  ticketId: "QA-1234",
  ticketDetails: {
    summary: "Implement user login functionality",
    description: "Add login form...",
    acceptanceCriteria: [...]
  },
  retryCount: 0
}

// After Test Generator
{
  ...previous,
  testPlan: {
    testCases: [
      { id: "TC-001", title: "Valid login", priority: "high" },
      { id: "TC-002", title: "Invalid email", priority: "high" },
      // ... 6 more
    ],
    coverage: 85
  }
}

// After Test Executor
{
  ...previous,
  testResults: {
    passed: 7,
    failed: 1,
    screenshots: ["/tmp/screenshot_1.png", ...]
  }
}

// Final State
{
  ...previous,
  pdfReportUrl: "https://qa-bucket.s3.amazonaws.com/reports/QA-1234_1234567890.pdf",
  manualReview: false  // Skipped (only 8 test cases)
}
```

### Key Takeaways from QA Example

1. **Conditional Routing**: Workflow adapts based on complexity and results
2. **Human-in-the-Loop**: Manual review for complex test plans (>10 cases)
3. **Retry Logic**: Auto-regenerates tests if failure rate > 50%
4. **State Evolution**: Complete state carries through all nodes
5. **Real-World Integration**: JIRA, Playwright, S3, PDF generation

### Comparison: Banking vs QA Workflow

```typescript
// Banking Loan Approval
workflow.addConditionalEdges('riskAssessment', (state) => {
  if (state.riskScore > 70) return 'approvalNode';
  if (state.riskScore > 40) return 'manualReview';
  return 'rejectNode';
});

// QA Test Automation
workflow.addConditionalEdges('testGenerator', (state) => {
  const complexity = state.testPlan.testCases.length;
  return complexity > 10 ? 'manualReview' : 'testExecutor';
});

// Similarities:
// ✓ Conditional routing based on data
// ✓ Human-in-the-loop for edge cases
// ✓ State management across nodes
// ✓ Error handling and retries

// Differences:
// • Banking: Risk-based routing (financial impact)
// • QA: Complexity-based routing (test coverage)
// • Banking: Regulatory compliance checkpoints
// • QA: Artifact management (screenshots, logs)
```

---

## 9. Best Practices Recap

### Do's ✅

1. **Define clear state interfaces** with TypeScript
2. **Keep nodes focused** on single responsibilities
3. **Use conditional routing** for decision points
4. **Implement retry logic** with max attempts
5. **Add human-in-the-loop** for critical decisions
6. **Log state transitions** for debugging
7. **Handle errors gracefully** in each node
8. **Test each node independently** before integration

### Don'ts ❌

1. **Don't create circular dependencies** without clear exit conditions
2. **Don't modify state directly** - return new state
3. **Don't skip error handling** in nodes
4. **Don't forget to set entry and end points**
5. **Don't put too much logic** in single nodes
6. **Don't ignore retry limits** (prevent infinite loops)

---

## 10. Quick Reference

### Workflow Setup

```typescript
// 1. Define state
interface State { ... }

// 2. Create graph
const workflow = new StateGraph<State>({ channels: {...} });

// 3. Add nodes
workflow.addNode('node_name', nodeFunction);

// 4. Add edges
workflow.addEdge('node1', 'node2');

// 5. Set entry
workflow.setEntryPoint('node1');

// 6. Compile
const app = workflow.compile();

// 7. Run
const result = await app.invoke(initialState);
```

### Common Patterns

```typescript
// Normal edge
workflow.addEdge('a', 'b');

// Conditional edge
workflow.addConditionalEdges('a', routerFn, { 'path1': 'b', 'path2': 'c' });

// End
workflow.addEdge('final', END);

// Entry
workflow.setEntryPoint('start');
```

---

**Next Steps:**
- Study `QA_Automation_Scenario.md` for complete 6-agent implementation
- Explore `Q51_Multi_Agent_Systems_Architecture.md` for multi-agent patterns
- Review `Architecture_Diagrams.md` for visual workflows
- Practice building conditional workflows with retry logic
- Experiment with checkpointing for long-running workflows

---

**Document Version**: 2.0  
**Last Updated**: November 2025  
**Scenarios**: QA Automation + ENBD Banking  
**Related Files**:
- QA_Automation_Scenario.md (complete QA architecture)
- Q51_Multi_Agent_Systems_Architecture.md (multi-agent implementations)
- Architecture_Diagrams.md (visual diagrams)
- Tools_in_Agent_Systems.md (tool integration guide)

---

*End of LangGraph Basics Document*

