# Q52: Agent Communication Patterns

## Question: How do agents communicate and coordinate in multi-agent systems?

---

## Answer:

Agent communication is essential for coordinating complex workflows. Agents need to pass messages, share state, delegate tasks, and handle handoffs seamlessly.

---

## Table of Contents

1. [Communication Patterns Overview](#communication-patterns-overview)
2. [Shared State Communication](#shared-state-communication)
3. [Message Passing](#message-passing)
4. [Agent Handoff Mechanisms](#agent-handoff-mechanisms)
5. [QA Automation Communication](#qa-automation-communication)
6. [Banking Communication Examples](#banking-communication-examples)
7. [Best Practices](#best-practices)

---

## 1. Communication Patterns Overview

### Types of Agent Communication

```
┌─────────────────────────────────────────────────────────────┐
│           AGENT COMMUNICATION PATTERNS                      │
└─────────────────────────────────────────────────────────────┘

1. SHARED STATE (LangGraph)
   ┌─────────┐
   │ Agent A │──┐
   └─────────┘  │
                ▼
   ┌──────────────────┐
   │  Shared State    │
   │  {data: {...}}   │
   └──────────────────┘
                ▲
   ┌─────────┐  │
   │ Agent B │──┘
   └─────────┘
   
2. MESSAGE PASSING (Direct)
   ┌─────────┐        ┌─────────┐
   │ Agent A │───────>│ Agent B │
   └─────────┘  msg   └─────────┘
   
3. EVENT BUS (Pub/Sub)
   ┌─────────┐
   │ Agent A │──┐
   └─────────┘  │
                ▼
   ┌──────────────────┐
   │    Event Bus     │
   └──────────────────┘
                ▲
   ┌─────────┐  │
   │ Agent B │──┘
   └─────────┘
   
4. HIERARCHICAL (Manager-Worker)
        ┌─────────┐
        │ Manager │
        └────┬────┘
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
   [W1]   [W2]   [W3]
```

---

## 2. Shared State Communication

### LangGraph Shared State Pattern

```typescript
import { StateGraph } from '@langchain/langgraph';
import { ChatOpenAI } from '@langchain/openai';

/**
 * Shared State Interface
 * All agents read from and write to this shared state
 */
interface SharedState {
  // Input data
  customerId: string;
  
  // Agent 1 writes, Agent 2 reads
  customerData?: {
    name: string;
    email: string;
    phoneNumber: string;
  };
  
  // Agent 2 writes, Agent 3 reads
  creditScore?: number;
  
  // Agent 3 writes, final output
  loanApproval?: {
    approved: boolean;
    amount: number;
    reason: string;
  };
  
  // Metadata for tracking
  agentHistory?: string[];
  errors?: string[];
}

/**
 * Agent 1: Customer Data Retriever
 * Writes: customerData
 */
async function customerDataAgent(state: SharedState): Promise<Partial<SharedState>> {
  console.log(`[Customer Data Agent] Processing customer ${state.customerId}`);
  
  // Simulate database lookup
  const customerData = {
    name: 'John Doe',
    email: 'john.doe@example.com',
    phoneNumber: '+971501234567',
  };
  
  return {
    customerData,
    agentHistory: [...(state.agentHistory || []), 'customerDataAgent'],
  };
}

/**
 * Agent 2: Credit Scorer
 * Reads: customerData
 * Writes: creditScore
 */
async function creditScorerAgent(state: SharedState): Promise<Partial<SharedState>> {
  console.log(`[Credit Scorer] Analyzing ${state.customerData?.name}`);
  
  // Read from shared state
  if (!state.customerData) {
    return {
      errors: [...(state.errors || []), 'Missing customer data'],
    };
  }
  
  // Simulate credit check
  const creditScore = Math.floor(Math.random() * 300) + 600; // 600-900
  
  return {
    creditScore,
    agentHistory: [...(state.agentHistory || []), 'creditScorerAgent'],
  };
}

/**
 * Agent 3: Loan Approver
 * Reads: customerData, creditScore
 * Writes: loanApproval
 */
async function loanApproverAgent(state: SharedState): Promise<Partial<SharedState>> {
  console.log(`[Loan Approver] Evaluating loan for ${state.customerData?.name}`);
  
  // Read from shared state
  const creditScore = state.creditScore || 0;
  
  let approval;
  if (creditScore >= 750) {
    approval = {
      approved: true,
      amount: 500000,
      reason: 'Excellent credit score',
    };
  } else if (creditScore >= 650) {
    approval = {
      approved: true,
      amount: 250000,
      reason: 'Good credit score',
    };
  } else {
    approval = {
      approved: false,
      amount: 0,
      reason: 'Credit score below minimum threshold',
    };
  }
  
  return {
    loanApproval: approval,
    agentHistory: [...(state.agentHistory || []), 'loanApproverAgent'],
  };
}

/**
 * Build workflow with shared state
 */
function createSharedStateWorkflow() {
  const workflow = new StateGraph<SharedState>({
    channels: {
      customerId: null,
      customerData: null,
      creditScore: null,
      loanApproval: null,
      agentHistory: null,
      errors: null,
    },
  });
  
  workflow.addNode('customerData', customerDataAgent);
  workflow.addNode('creditScorer', creditScorerAgent);
  workflow.addNode('loanApprover', loanApproverAgent);
  
  workflow.setEntryPoint('customerData');
  workflow.addEdge('customerData', 'creditScorer');
  workflow.addEdge('creditScorer', 'loanApprover');
  
  return workflow.compile();
}

// Usage
async function runSharedStateExample() {
  const app = createSharedStateWorkflow();
  
  const result = await app.invoke({
    customerId: 'CUST-12345',
    agentHistory: [],
    errors: [],
  });
  
  console.log('\n=== Shared State Result ===');
  console.log('Customer:', result.customerData?.name);
  console.log('Credit Score:', result.creditScore);
  console.log('Approval:', result.loanApproval);
  console.log('Agent Flow:', result.agentHistory);
}
```

### Key Benefits of Shared State

```typescript
/**
 * Benefits of Shared State Pattern
 */

// ✅ Single source of truth
const state = {
  ticketId: 'QA-1234',
  testResults: { passed: 7, failed: 1 },
};

// All agents see the same data
function agent1(state) {
  console.log(state.ticketId); // QA-1234
}

function agent2(state) {
  console.log(state.ticketId); // Same QA-1234
}

// ✅ Automatic state merging
function agentA(state) {
  return { dataA: 'value' }; // Merged into state
}

function agentB(state) {
  return { dataB: 'value' }; // Also merged
}
// Final state has both dataA and dataB

// ✅ No explicit message passing needed
// Agents don't need to know about each other
```

---

## 3. Message Passing

### Direct Message Passing Pattern

```typescript
import { ChatOpenAI } from '@langchain/openai';

/**
 * Message Interface
 */
interface Message {
  from: string;
  to: string;
  type: 'request' | 'response' | 'notification';
  payload: any;
  timestamp: Date;
  messageId: string;
}

/**
 * Message Bus for Agent Communication
 */
class MessageBus {
  private messages: Message[] = [];
  private subscribers: Map<string, (msg: Message) => void> = new Map();
  
  send(message: Message) {
    console.log(`[MessageBus] ${message.from} → ${message.to}: ${message.type}`);
    this.messages.push(message);
    
    // Deliver to subscriber
    const handler = this.subscribers.get(message.to);
    if (handler) {
      handler(message);
    }
  }
  
  subscribe(agentName: string, handler: (msg: Message) => void) {
    this.subscribers.set(agentName, handler);
  }
  
  getHistory(): Message[] {
    return [...this.messages];
  }
}

/**
 * Base Agent with Message Passing
 */
class BaseAgent {
  constructor(
    public name: string,
    protected messageBus: MessageBus,
    protected model: ChatOpenAI
  ) {
    // Subscribe to messages for this agent
    this.messageBus.subscribe(this.name, (msg) => this.handleMessage(msg));
  }
  
  protected sendMessage(to: string, type: Message['type'], payload: any) {
    const message: Message = {
      from: this.name,
      to,
      type,
      payload,
      timestamp: new Date(),
      messageId: `msg_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
    };
    
    this.messageBus.send(message);
  }
  
  protected async handleMessage(message: Message) {
    console.log(`[${this.name}] Received ${message.type} from ${message.from}`);
    // Override in subclasses
  }
}

/**
 * QA Example: JIRA Agent
 */
class JiraAgent extends BaseAgent {
  async fetchTicket(ticketId: string) {
    console.log(`[${this.name}] Fetching ticket ${ticketId}...`);
    
    const ticketData = {
      ticketId,
      summary: 'Implement login functionality',
      description: 'Add email/password login form',
      acceptanceCriteria: ['Validation', 'Error handling', 'Redirect'],
    };
    
    // Send ticket data to Test Generator
    this.sendMessage('TestGeneratorAgent', 'request', {
      action: 'generateTests',
      ticketData,
    });
    
    return ticketData;
  }
  
  protected async handleMessage(message: Message) {
    await super.handleMessage(message);
    
    if (message.type === 'notification' && message.payload.action === 'testsComplete') {
      console.log(`[${this.name}] Tests completed, updating JIRA...`);
      // Update JIRA with results
    }
  }
}

/**
 * QA Example: Test Generator Agent
 */
class TestGeneratorAgent extends BaseAgent {
  protected async handleMessage(message: Message) {
    await super.handleMessage(message);
    
    if (message.type === 'request' && message.payload.action === 'generateTests') {
      const ticketData = message.payload.ticketData;
      
      console.log(`[${this.name}] Generating tests for ${ticketData.ticketId}...`);
      
      const testPlan = {
        testCases: [
          { id: 'TC-001', title: 'Valid login' },
          { id: 'TC-002', title: 'Invalid email' },
          { id: 'TC-003', title: 'Empty password' },
        ],
        coverage: 85,
      };
      
      // Send test plan to Test Executor
      this.sendMessage('TestExecutorAgent', 'request', {
        action: 'executeTests',
        testPlan,
        ticketId: ticketData.ticketId,
      });
    }
  }
}

/**
 * QA Example: Test Executor Agent
 */
class TestExecutorAgent extends BaseAgent {
  protected async handleMessage(message: Message) {
    await super.handleMessage(message);
    
    if (message.type === 'request' && message.payload.action === 'executeTests') {
      const { testPlan, ticketId } = message.payload;
      
      console.log(`[${this.name}] Executing ${testPlan.testCases.length} tests...`);
      
      const results = {
        passed: 2,
        failed: 1,
        screenshots: ['/tmp/screenshot_1.png', '/tmp/screenshot_2.png'],
      };
      
      // Notify JIRA Agent
      this.sendMessage('JiraAgent', 'notification', {
        action: 'testsComplete',
        ticketId,
        results,
      });
      
      // Send results to Report Generator
      this.sendMessage('ReportGeneratorAgent', 'request', {
        action: 'generateReport',
        ticketId,
        testPlan,
        results,
      });
    }
  }
}

/**
 * QA Example: Report Generator Agent
 */
class ReportGeneratorAgent extends BaseAgent {
  protected async handleMessage(message: Message) {
    await super.handleMessage(message);
    
    if (message.type === 'request' && message.payload.action === 'generateReport') {
      const { ticketId, testPlan, results } = message.payload;
      
      console.log(`[${this.name}] Generating PDF report for ${ticketId}...`);
      
      const reportUrl = `https://s3.amazonaws.com/reports/${ticketId}.pdf`;
      
      // Notify JIRA Agent with report URL
      this.sendMessage('JiraAgent', 'notification', {
        action: 'reportReady',
        ticketId,
        reportUrl,
      });
    }
  }
}

/**
 * Usage Example
 */
async function runMessagePassingExample() {
  const messageBus = new MessageBus();
  const model = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });
  
  // Create agents
  const jiraAgent = new JiraAgent('JiraAgent', messageBus, model);
  const testGeneratorAgent = new TestGeneratorAgent('TestGeneratorAgent', messageBus, model);
  const testExecutorAgent = new TestExecutorAgent('TestExecutorAgent', messageBus, model);
  const reportGeneratorAgent = new ReportGeneratorAgent('ReportGeneratorAgent', messageBus, model);
  
  // Start workflow
  await jiraAgent.fetchTicket('QA-1234');
  
  // Wait for async message processing
  await new Promise((resolve) => setTimeout(resolve, 1000));
  
  console.log('\n=== Message History ===');
  messageBus.getHistory().forEach((msg, idx) => {
    console.log(`${idx + 1}. ${msg.from} → ${msg.to}: ${msg.type} (${msg.payload.action})`);
  });
}
```

---

## 4. Agent Handoff Mechanisms

### Explicit Handoff Pattern

```typescript
/**
 * Handoff Interface
 */
interface AgentHandoff {
  fromAgent: string;
  toAgent: string;
  context: any;
  reason: string;
  timestamp: Date;
}

/**
 * Banking Example: Customer Service Routing
 */
class CustomerServiceRouter {
  private agents: Map<string, any> = new Map();
  private handoffHistory: AgentHandoff[] = [];
  
  registerAgent(name: string, agent: any) {
    this.agents.set(name, agent);
  }
  
  async handoff(fromAgent: string, toAgent: string, context: any, reason: string) {
    console.log(`[Handoff] ${fromAgent} → ${toAgent}: ${reason}`);
    
    const handoff: AgentHandoff = {
      fromAgent,
      toAgent,
      context,
      reason,
      timestamp: new Date(),
    };
    
    this.handoffHistory.push(handoff);
    
    // Transfer context to next agent
    const targetAgent = this.agents.get(toAgent);
    if (targetAgent) {
      return await targetAgent.process(context);
    }
    
    throw new Error(`Agent ${toAgent} not found`);
  }
  
  getHandoffHistory(): AgentHandoff[] {
    return [...this.handoffHistory];
  }
}

/**
 * General Customer Service Agent
 */
class GeneralSupportAgent {
  constructor(private router: CustomerServiceRouter) {}
  
  async process(query: string) {
    console.log('[General Support] Processing query...');
    
    // Analyze query to determine specialization needed
    if (query.toLowerCase().includes('loan')) {
      return this.router.handoff(
        'GeneralSupport',
        'LoanSpecialist',
        { query, customerTier: 'premium' },
        'Query requires loan expertise'
      );
    } else if (query.toLowerCase().includes('fraud')) {
      return this.router.handoff(
        'GeneralSupport',
        'FraudSpecialist',
        { query, priority: 'high' },
        'Potential fraud issue detected'
      );
    }
    
    return { handled: true, response: 'General inquiry handled' };
  }
}

/**
 * Loan Specialist Agent
 */
class LoanSpecialistAgent {
  async process(context: any) {
    console.log('[Loan Specialist] Handling loan inquiry...');
    
    const { query, customerTier } = context;
    
    // Complex loan calculation or advice
    const response = {
      handled: true,
      recommendation: 'Personal loan up to AED 500,000',
      interestRate: customerTier === 'premium' ? 3.5 : 4.2,
      referenceNumber: `LOAN-${Date.now()}`,
    };
    
    return response;
  }
}

/**
 * Fraud Specialist Agent
 */
class FraudSpecialistAgent {
  constructor(private router: CustomerServiceRouter) {}
  
  async process(context: any) {
    console.log('[Fraud Specialist] Investigating potential fraud...');
    
    const { query, priority } = context;
    
    // Perform fraud analysis
    const riskLevel = Math.random() > 0.7 ? 'high' : 'low';
    
    if (riskLevel === 'high') {
      // Escalate to manager
      return this.router.handoff(
        'FraudSpecialist',
        'ManagerAgent',
        { ...context, riskLevel, analysis: 'Suspicious activity detected' },
        'High-risk fraud case requires manager approval'
      );
    }
    
    return {
      handled: true,
      riskLevel,
      action: 'Case closed - no fraud detected',
    };
  }
}

/**
 * Manager Agent (Final Escalation)
 */
class ManagerAgent {
  async process(context: any) {
    console.log('[Manager] Reviewing escalated case...');
    
    return {
      handled: true,
      managerDecision: 'Account temporarily suspended pending investigation',
      caseId: `MGR-${Date.now()}`,
      followUpRequired: true,
    };
  }
}

/**
 * Usage Example
 */
async function runHandoffExample() {
  const router = new CustomerServiceRouter();
  
  // Register agents
  const generalAgent = new GeneralSupportAgent(router);
  const loanAgent = new LoanSpecialistAgent();
  const fraudAgent = new FraudSpecialistAgent(router);
  const managerAgent = new ManagerAgent();
  
  router.registerAgent('GeneralSupport', generalAgent);
  router.registerAgent('LoanSpecialist', loanAgent);
  router.registerAgent('FraudSpecialist', fraudAgent);
  router.registerAgent('ManagerAgent', managerAgent);
  
  // Test 1: Loan inquiry (1 handoff)
  console.log('\n=== Test 1: Loan Inquiry ===');
  const result1 = await generalAgent.process('I want to apply for a loan');
  console.log('Result:', result1);
  
  // Test 2: Fraud inquiry (2 handoffs - might escalate to manager)
  console.log('\n=== Test 2: Fraud Inquiry ===');
  const result2 = await generalAgent.process('I see unauthorized transactions on my account');
  console.log('Result:', result2);
  
  console.log('\n=== Handoff History ===');
  router.getHandoffHistory().forEach((handoff, idx) => {
    console.log(`${idx + 1}. ${handoff.fromAgent} → ${handoff.toAgent}: ${handoff.reason}`);
  });
}
```

---

## 5. QA Automation Communication

### Complete QA Agent Communication Flow

```typescript
import { StateGraph } from '@langchain/langgraph';

/**
 * QA Automation with Agent Communication Tracking
 */
interface QAState {
  ticketId: string;
  
  // Communication metadata
  agentCommunication: {
    from: string;
    to: string;
    data: string;
    timestamp: Date;
  }[];
  
  // Data from agents
  ticketDetails?: any;
  testPlan?: any;
  testResults?: any;
  artifacts?: any;
  pdfReport?: any;
}

/**
 * Communication Helper
 */
function logCommunication(
  state: QAState,
  from: string,
  to: string,
  data: string
): QAState['agentCommunication'] {
  const communication = {
    from,
    to,
    data,
    timestamp: new Date(),
  };
  
  console.log(`[Communication] ${from} → ${to}: ${data}`);
  
  return [...(state.agentCommunication || []), communication];
}

/**
 * JIRA Reader Agent with Communication
 */
async function jiraReaderNode(state: QAState): Promise<Partial<QAState>> {
  const ticketDetails = { summary: 'Login feature', description: '...' };
  
  return {
    ticketDetails,
    agentCommunication: logCommunication(
      state,
      'JiraReader',
      'TestGenerator',
      `Ticket details for ${state.ticketId}: ${JSON.stringify(ticketDetails).substring(0, 50)}...`
    ),
  };
}

/**
 * Test Generator Agent with Communication
 */
async function testGeneratorNode(state: QAState): Promise<Partial<QAState>> {
  // Read data from previous agent (JIRA Reader)
  console.log(`[TestGenerator] Received ticket: ${state.ticketDetails?.summary}`);
  
  const testPlan = {
    testCases: [{ id: 'TC-001' }, { id: 'TC-002' }],
    coverage: 85,
  };
  
  return {
    testPlan,
    agentCommunication: logCommunication(
      state,
      'TestGenerator',
      'TestExecutor',
      `Test plan ready: ${testPlan.testCases.length} test cases, ${testPlan.coverage}% coverage`
    ),
  };
}

/**
 * Test Executor Agent with Communication
 */
async function testExecutorNode(state: QAState): Promise<Partial<QAState>> {
  // Read data from previous agent (Test Generator)
  console.log(`[TestExecutor] Executing ${state.testPlan?.testCases.length} tests...`);
  
  const testResults = {
    passed: 7,
    failed: 1,
    screenshots: ['/tmp/screenshot_1.png'],
  };
  
  return {
    testResults,
    agentCommunication: logCommunication(
      state,
      'TestExecutor',
      'ArtifactManager',
      `Tests complete: ${testResults.passed} passed, ${testResults.failed} failed`
    ),
  };
}

/**
 * Artifact Manager Agent with Communication
 */
async function artifactManagerNode(state: QAState): Promise<Partial<QAState>> {
  // Read data from previous agent (Test Executor)
  console.log(`[ArtifactManager] Uploading ${state.testResults?.screenshots.length} screenshots...`);
  
  const artifacts = {
    s3Urls: ['https://s3.amazonaws.com/screenshots/1.png'],
  };
  
  return {
    artifacts,
    agentCommunication: logCommunication(
      state,
      'ArtifactManager',
      'ReportGenerator',
      `Artifacts uploaded: ${artifacts.s3Urls.length} files`
    ),
  };
}

/**
 * Report Generator Agent with Communication
 */
async function reportGeneratorNode(state: QAState): Promise<Partial<QAState>> {
  // Read data from all previous agents
  console.log(`[ReportGenerator] Compiling report for ${state.ticketId}...`);
  console.log(`  - Ticket: ${state.ticketDetails?.summary}`);
  console.log(`  - Tests: ${state.testPlan?.testCases.length} cases`);
  console.log(`  - Results: ${state.testResults?.passed}/${state.testResults?.failed}`);
  console.log(`  - Artifacts: ${state.artifacts?.s3Urls.length} files`);
  
  const pdfReport = {
    url: `https://s3.amazonaws.com/reports/${state.ticketId}.pdf`,
  };
  
  return {
    pdfReport,
    agentCommunication: logCommunication(
      state,
      'ReportGenerator',
      'END',
      `Report generated: ${pdfReport.url}`
    ),
  };
}

/**
 * Build workflow with communication tracking
 */
function createQACommunicationWorkflow() {
  const workflow = new StateGraph<QAState>({
    channels: {
      ticketId: null,
      agentCommunication: null,
      ticketDetails: null,
      testPlan: null,
      testResults: null,
      artifacts: null,
      pdfReport: null,
    },
  });
  
  workflow.addNode('jiraReader', jiraReaderNode);
  workflow.addNode('testGenerator', testGeneratorNode);
  workflow.addNode('testExecutor', testExecutorNode);
  workflow.addNode('artifactManager', artifactManagerNode);
  workflow.addNode('reportGenerator', reportGeneratorNode);
  
  workflow.setEntryPoint('jiraReader');
  workflow.addEdge('jiraReader', 'testGenerator');
  workflow.addEdge('testGenerator', 'testExecutor');
  workflow.addEdge('testExecutor', 'artifactManager');
  workflow.addEdge('artifactManager', 'reportGenerator');
  
  return workflow.compile();
}

/**
 * Usage
 */
async function runQACommunicationExample() {
  const app = createQACommunicationWorkflow();
  
  const result = await app.invoke({
    ticketId: 'QA-1234',
    agentCommunication: [],
  });
  
  console.log('\n=== Communication Timeline ===');
  result.agentCommunication?.forEach((comm, idx) => {
    console.log(`${idx + 1}. [${comm.timestamp.toLocaleTimeString()}] ${comm.from} → ${comm.to}`);
    console.log(`   ${comm.data}`);
  });
}
```

---

## 6. Banking Communication Examples

### Loan Processing Agent Communication

```typescript
/**
 * Banking: Loan Processing with Inter-Agent Communication
 */
interface LoanState {
  applicationId: string;
  
  // Communication log
  communications: {
    from: string;
    to: string;
    message: string;
    timestamp: Date;
  }[];
  
  // Agent data
  customerData?: any;
  creditCheck?: any;
  riskAssessment?: any;
  approval?: any;
}

async function creditAgentNode(state: LoanState): Promise<Partial<LoanState>> {
  console.log(`[Credit Agent] Assessing credit for ${state.customerData?.name}...`);
  
  const creditCheck = {
    score: 750,
    history: 'Clean',
    recommendation: 'Approve',
  };
  
  // Communicate to Risk Agent
  const communication = {
    from: 'CreditAgent',
    to: 'RiskAgent',
    message: `Credit score: ${creditCheck.score}, recommending approval`,
    timestamp: new Date(),
  };
  
  return {
    creditCheck,
    communications: [...(state.communications || []), communication],
  };
}

async function riskAgentNode(state: LoanState): Promise<Partial<LoanState>> {
  // Read communication from Credit Agent
  const lastComm = state.communications?.[state.communications.length - 1];
  console.log(`[Risk Agent] Received from ${lastComm?.from}: ${lastComm?.message}`);
  
  const riskAssessment = {
    riskLevel: 'Low',
    factors: ['Good credit', 'Stable income'],
  };
  
  // Communicate to Approval Agent
  const communication = {
    from: 'RiskAgent',
    to: 'ApprovalAgent',
    message: `Risk: ${riskAssessment.riskLevel}, factors: ${riskAssessment.factors.join(', ')}`,
    timestamp: new Date(),
  };
  
  return {
    riskAssessment,
    communications: [...(state.communications || []), communication],
  };
}

async function approvalAgentNode(state: LoanState): Promise<Partial<LoanState>> {
  // Read all previous communications
  console.log('[Approval Agent] Reviewing all agent inputs...');
  state.communications?.forEach((comm) => {
    console.log(`  ${comm.from}: ${comm.message}`);
  });
  
  const approval = {
    approved: true,
    amount: 500000,
    interestRate: 3.5,
  };
  
  const communication = {
    from: 'ApprovalAgent',
    to: 'Customer',
    message: `Loan approved: AED ${approval.amount} at ${approval.interestRate}%`,
    timestamp: new Date(),
  };
  
  return {
    approval,
    communications: [...(state.communications || []), communication],
  };
}
```

---

## 7. Best Practices

### Communication Patterns Comparison

```typescript
/**
 * When to Use Each Pattern
 */

// ✅ SHARED STATE (LangGraph)
// Use when:
// - Agents need access to all previous data
// - State evolution is important
// - Sequential or conditional workflows
// - Single workflow execution

// Example: QA automation, loan processing
const sharedStateWorkflow = new StateGraph<State>({ channels: {...} });

// ✅ MESSAGE PASSING
// Use when:
// - Agents are independent services
// - Async communication needed
// - Loose coupling preferred
// - Event-driven architecture

// Example: Microservices, distributed agents
const messageBus = new MessageBus();
agent1.send(agent2, message);

// ✅ HANDOFF
// Use when:
// - Specialization required
// - Escalation workflows
// - Context must be preserved
// - Human-in-the-loop

// Example: Customer service routing
router.handoff('GeneralAgent', 'SpecialistAgent', context);

// ✅ EVENT BUS (Pub/Sub)
// Use when:
// - One-to-many communication
// - Decoupled agents
// - Event-driven reactions
// - Broadcasting updates

// Example: Notification systems
eventBus.publish('testComplete', { ticketId, results });
```

### Communication Anti-Patterns

```typescript
/**
 * ❌ DON'T: Circular dependencies without exit condition
 */
// Bad: Agent A calls Agent B calls Agent A indefinitely
function agentA(state) {
  return callAgentB(state);
}
function agentB(state) {
  return callAgentA(state); // ❌ Infinite loop!
}

// ✅ Good: Use conditional routing with exit
workflow.addConditionalEdges('agentA', (state) => {
  return state.iteration < 3 ? 'agentB' : END;
});

/**
 * ❌ DON'T: Tight coupling between agents
 */
// Bad: Agent directly imports and calls another agent
import { TestExecutor } from './test-executor';
function testGenerator() {
  const executor = new TestExecutor();
  executor.execute(); // ❌ Tight coupling
}

// ✅ Good: Use shared state or message bus
function testGenerator(state) {
  return { testPlan: {...} }; // Next agent reads from state
}

/**
 * ❌ DON'T: Missing error handling in communication
 */
// Bad: No error handling
function agent(state) {
  return callExternalApi(); // ❌ What if it fails?
}

// ✅ Good: Handle communication failures
function agent(state) {
  try {
    return callExternalApi();
  } catch (error) {
    return { error: `Communication failed: ${error.message}` };
  }
}
```

### Best Practices Summary

```
┌─────────────────────────────────────────────────────────────┐
│           AGENT COMMUNICATION BEST PRACTICES                │
└─────────────────────────────────────────────────────────────┘

✅ DO:
━━━━━
1. Use shared state for sequential workflows
2. Log all agent-to-agent communications
3. Include metadata (timestamp, sender, receiver)
4. Handle communication failures gracefully
5. Validate data received from other agents
6. Use typed interfaces for messages
7. Implement timeout mechanisms
8. Provide clear handoff context

❌ DON'T:
━━━━━━━━
1. Create circular dependencies without exits
2. Tightly couple agents together
3. Skip error handling in communication
4. Forget to track communication history
5. Pass sensitive data without encryption
6. Ignore message ordering when it matters
7. Create complex communication patterns unnecessarily
8. Forget to document communication flows
```

---

## Summary

### Key Takeaways

1. **Shared State (LangGraph)**: Best for sequential workflows where agents need access to all previous data
2. **Message Passing**: Best for distributed, loosely-coupled agent systems
3. **Agent Handoff**: Best for specialization and escalation workflows
4. **Event Bus**: Best for one-to-many communication and event-driven architectures

### Pattern Selection Guide

| Use Case | Pattern | Why? |
|----------|---------|------|
| QA Automation | Shared State | Sequential workflow, state evolution important |
| Loan Processing | Shared State | Multiple decision points, state tracking needed |
| Customer Service | Handoff | Specialization, escalation, context preservation |
| Microservices | Message Passing | Distributed, async, loose coupling |
| Notifications | Event Bus | One-to-many, broadcasting |

### Real-World Applications

**QA Automation:**
- JIRA Reader → Test Generator: Ticket details
- Test Generator → Test Executor: Test plan
- Test Executor → Artifact Manager: Test results
- Artifact Manager → Report Generator: S3 URLs

**Banking (ENBD):**
- General Support → Loan Specialist: Customer query handoff
- Credit Agent → Risk Agent: Credit score communication
- Risk Agent → Approval Agent: Risk assessment
- Fraud Agent → Manager: High-risk escalation

---

**Related Files:**
- `Q51_Multi_Agent_Systems_Architecture.md` - Multi-agent patterns
- `Q53_Conversational_Memory_Management.md` - Memory in communication
- `QA_Automation_Scenario.md` - Complete QA implementation
- `Architecture_Diagrams.md` - Visual communication flows

---

**Document Version**: 1.0  
**Last Updated**: November 2025  
**For**: GenAI Interview Preparation

---

*End of Agent Communication Patterns Document*
