# Q51: Multi-Agent Systems Architecture

## Question: How do you design and implement multi-agent systems for complex GenAI workflows?

---

## Answer:

Multi-agent systems allow multiple specialized AI agents to collaborate on complex tasks that would be difficult for a single agent. Each agent has a specific role, expertise, and can communicate with other agents to achieve a common goal.

---

## 1. Basic Multi-Agent Architecture

### Simple Sequential Multi-Agent System

```typescript
import { ChatOpenAI } from '@langchain/openai';

/**
 * Base Agent Interface
 */
interface Agent {
  name: string;
  role: string;
  execute(input: string, context?: any): Promise<AgentResult>;
}

interface AgentResult {
  agentName: string;
  output: string;
  metadata?: any;
  nextAgent?: string;
}

/**
 * Research Agent - Gathers information
 */
class ResearchAgent implements Agent {
  name = 'ResearchAgent';
  role = 'Research and Information Gathering';
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.3,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }

  async execute(input: string, context?: any): Promise<AgentResult> {
    console.log(`\n[${this.name}] Starting research...`);

    const prompt = `You are a research specialist at Emirates NBD bank.
    
Task: Research the following topic and provide comprehensive information.

Topic: ${input}

Context: ${context ? JSON.stringify(context) : 'None'}

Provide:
1. Key facts and data
2. Relevant regulations (UAE Central Bank)
3. Industry best practices
4. Risk factors
5. Recommendations

Format as structured JSON.`;

    const response = await this.model.call(prompt);

    return {
      agentName: this.name,
      output: response.content,
      metadata: {
        timestamp: new Date(),
        tokensUsed: response.content.length / 4
      },
      nextAgent: 'AnalysisAgent'
    };
  }
}

/**
 * Analysis Agent - Analyzes research findings
 */
class AnalysisAgent implements Agent {
  name = 'AnalysisAgent';
  role = 'Data Analysis and Insights';
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.2,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }

  async execute(input: string, context?: any): Promise<AgentResult> {
    console.log(`\n[${this.name}] Analyzing data...`);

    const prompt = `You are a data analysis specialist at Emirates NBD bank.

Research Data: ${input}

Previous Context: ${context ? JSON.stringify(context) : 'None'}

Analyze the research and provide:
1. Key insights
2. Risk assessment (High/Medium/Low)
3. Financial impact analysis
4. Regulatory compliance check
5. Strategic recommendations

Format as structured JSON with scores and ratings.`;

    const response = await this.model.call(prompt);

    return {
      agentName: this.name,
      output: response.content,
      metadata: {
        timestamp: new Date(),
        analysisScore: Math.random() * 100
      },
      nextAgent: 'ReportAgent'
    };
  }
}

/**
 * Report Agent - Creates final report
 */
class ReportAgent implements Agent {
  name = 'ReportAgent';
  role = 'Report Generation';
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.1,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }

  async execute(input: string, context?: any): Promise<AgentResult> {
    console.log(`\n[${this.name}] Generating report...`);

    const prompt = `You are a report writing specialist at Emirates NBD bank.

Analysis Results: ${input}

Full Context: ${context ? JSON.stringify(context) : 'None'}

Create a comprehensive executive report with:
1. Executive Summary
2. Detailed Findings
3. Risk Assessment
4. Financial Implications
5. Recommendations
6. Next Steps

Format professionally for senior management.`;

    const response = await this.model.call(prompt);

    return {
      agentName: this.name,
      output: response.content,
      metadata: {
        timestamp: new Date(),
        reportLength: response.content.length
      },
      nextAgent: null // Final agent
    };
  }
}

/**
 * Simple Sequential Orchestrator
 */
class SequentialOrchestrator {
  private agents: Map<string, Agent> = new Map();
  private executionLog: any[] = [];

  registerAgent(agent: Agent) {
    this.agents.set(agent.name, agent);
    console.log(`✅ Registered agent: ${agent.name} (${agent.role})`);
  }

  async execute(initialInput: string, startAgent: string): Promise<any> {
    console.log(`\n🚀 Starting multi-agent workflow...`);
    console.log(`📝 Initial task: ${initialInput}\n`);

    let currentAgentName = startAgent;
    let currentInput = initialInput;
    const context: any = {};

    while (currentAgentName) {
      const agent = this.agents.get(currentAgentName);
      
      if (!agent) {
        throw new Error(`Agent ${currentAgentName} not found`);
      }

      // Execute agent
      const startTime = Date.now();
      const result = await agent.execute(currentInput, context);
      const duration = Date.now() - startTime;

      // Log execution
      this.executionLog.push({
        agent: result.agentName,
        duration: `${duration}ms`,
        timestamp: new Date(),
        outputLength: result.output.length
      });

      console.log(`✅ ${result.agentName} completed in ${duration}ms`);

      // Update context
      context[result.agentName] = result.output;

      // Prepare for next agent
      currentInput = result.output;
      currentAgentName = result.nextAgent || null;
    }

    console.log(`\n✅ Multi-agent workflow completed!`);
    console.log(`📊 Total agents executed: ${this.executionLog.length}`);

    return {
      finalOutput: currentInput,
      executionLog: this.executionLog,
      fullContext: context
    };
  }

  getExecutionLog() {
    return this.executionLog;
  }
}

// Usage Example
async function runSimpleMultiAgent() {
  const orchestrator = new SequentialOrchestrator();

  // Register agents
  orchestrator.registerAgent(new ResearchAgent());
  orchestrator.registerAgent(new AnalysisAgent());
  orchestrator.registerAgent(new ReportAgent());

  // Execute workflow
  const result = await orchestrator.execute(
    'Analyze the impact of implementing digital KYC for ENBD retail customers',
    'ResearchAgent'
  );

  console.log('\n📄 Final Report:\n', result.finalOutput);
  console.log('\n📊 Execution Summary:', result.executionLog);
}
```

---

## 2. LangGraph Multi-Agent System

### Advanced Agent Workflow with State Management

```typescript
import { StateGraph, END } from '@langchain/langgraph';
import { ChatOpenAI } from '@langchain/openai';

/**
 * Loan Processing Multi-Agent System using LangGraph
 */

interface LoanApplicationState {
  applicationId: string;
  customerData: any;
  creditScore?: number;
  riskAssessment?: string;
  loanDecision?: string;
  approvalAmount?: number;
  currentAgent: string;
  history: string[];
}

/**
 * Credit Assessment Agent
 */
class CreditAssessmentAgent {
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }

  async assess(state: LoanApplicationState): Promise<Partial<LoanApplicationState>> {
    console.log(`\n[CreditAssessmentAgent] Assessing customer credit...`);

    const prompt = `You are a credit assessment specialist at Emirates NBD.

Customer Data:
${JSON.stringify(state.customerData, null, 2)}

Analyze and provide:
1. Credit score (300-850)
2. Credit history summary
3. Debt-to-income ratio assessment
4. Payment history evaluation

Return JSON: { creditScore: number, summary: string, recommendation: string }`;

    const response = await this.model.call(prompt);
    
    try {
      const assessment = JSON.parse(response.content);
      
      return {
        creditScore: assessment.creditScore,
        currentAgent: 'RiskAssessmentAgent',
        history: [
          ...state.history,
          `CreditAssessmentAgent: Credit Score = ${assessment.creditScore}`
        ]
      };
    } catch (error) {
      return {
        creditScore: 650, // Default
        currentAgent: 'RiskAssessmentAgent',
        history: [...state.history, 'CreditAssessmentAgent: Completed']
      };
    }
  }
}

/**
 * Risk Assessment Agent
 */
class RiskAssessmentAgent {
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }

  async assess(state: LoanApplicationState): Promise<Partial<LoanApplicationState>> {
    console.log(`\n[RiskAssessmentAgent] Assessing loan risk...`);

    const prompt = `You are a risk assessment specialist at Emirates NBD.

Application ID: ${state.applicationId}
Credit Score: ${state.creditScore}
Customer Data: ${JSON.stringify(state.customerData)}

Assess risk level based on:
1. Credit score
2. Employment stability
3. Existing debt obligations
4. Loan-to-value ratio
5. Market conditions

Return JSON: { riskLevel: "LOW"|"MEDIUM"|"HIGH", details: string, recommendation: string }`;

    const response = await this.model.call(prompt);
    
    try {
      const risk = JSON.parse(response.content);
      
      return {
        riskAssessment: risk.riskLevel,
        currentAgent: 'ApprovalAgent',
        history: [
          ...state.history,
          `RiskAssessmentAgent: Risk Level = ${risk.riskLevel}`
        ]
      };
    } catch (error) {
      return {
        riskAssessment: 'MEDIUM',
        currentAgent: 'ApprovalAgent',
        history: [...state.history, 'RiskAssessmentAgent: Completed']
      };
    }
  }
}

/**
 * Approval Agent
 */
class ApprovalAgent {
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });
  }

  async approve(state: LoanApplicationState): Promise<Partial<LoanApplicationState>> {
    console.log(`\n[ApprovalAgent] Making final decision...`);

    const requestedAmount = state.customerData.loanAmount;
    const creditScore = state.creditScore || 650;
    const riskLevel = state.riskAssessment || 'MEDIUM';

    // Decision logic
    let decision = 'REJECTED';
    let approvalAmount = 0;

    if (creditScore >= 750 && riskLevel === 'LOW') {
      decision = 'APPROVED';
      approvalAmount = requestedAmount;
    } else if (creditScore >= 680 && riskLevel !== 'HIGH') {
      decision = 'APPROVED';
      approvalAmount = Math.floor(requestedAmount * 0.8); // 80% of requested
    } else if (creditScore >= 620 && riskLevel === 'LOW') {
      decision = 'APPROVED';
      approvalAmount = Math.floor(requestedAmount * 0.6); // 60% of requested
    }

    return {
      loanDecision: decision,
      approvalAmount,
      currentAgent: 'END',
      history: [
        ...state.history,
        `ApprovalAgent: Decision = ${decision}, Amount = AED ${approvalAmount.toLocaleString()}`
      ]
    };
  }
}

/**
 * LangGraph Loan Processing System
 */
class LoanProcessingGraph {
  private graph: StateGraph;
  private creditAgent: CreditAssessmentAgent;
  private riskAgent: RiskAssessmentAgent;
  private approvalAgent: ApprovalAgent;

  constructor() {
    this.creditAgent = new CreditAssessmentAgent();
    this.riskAgent = new RiskAssessmentAgent();
    this.approvalAgent = new ApprovalAgent();

    this.graph = new StateGraph({
      channels: {
        applicationId: null,
        customerData: null,
        creditScore: null,
        riskAssessment: null,
        loanDecision: null,
        approvalAmount: null,
        currentAgent: null,
        history: null
      }
    });

    this.buildGraph();
  }

  private buildGraph() {
    // Add nodes (agents)
    this.graph.addNode('creditAssessment', async (state: LoanApplicationState) => {
      return await this.creditAgent.assess(state);
    });

    this.graph.addNode('riskAssessment', async (state: LoanApplicationState) => {
      return await this.riskAgent.assess(state);
    });

    this.graph.addNode('approval', async (state: LoanApplicationState) => {
      return await this.approvalAgent.approve(state);
    });

    // Define edges (workflow)
    this.graph.addEdge('creditAssessment', 'riskAssessment');
    this.graph.addEdge('riskAssessment', 'approval');
    this.graph.addEdge('approval', END);

    // Set entry point
    this.graph.setEntryPoint('creditAssessment');
  }

  async process(applicationData: any): Promise<LoanApplicationState> {
    console.log(`\n🏦 ENBD Loan Processing - Multi-Agent System`);
    console.log(`📋 Application ID: ${applicationData.applicationId}`);
    console.log(`💰 Requested Amount: AED ${applicationData.loanAmount.toLocaleString()}\n`);

    const initialState: LoanApplicationState = {
      applicationId: applicationData.applicationId,
      customerData: applicationData,
      currentAgent: 'creditAssessment',
      history: ['Application received']
    };

    const compiledGraph = this.graph.compile();
    const result = await compiledGraph.invoke(initialState);

    console.log(`\n✅ Processing Complete!`);
    console.log(`📊 Final Decision: ${result.loanDecision}`);
    console.log(`💵 Approval Amount: AED ${result.approvalAmount?.toLocaleString() || 0}`);
    console.log(`\n📝 Processing History:`);
    result.history.forEach((entry, index) => {
      console.log(`   ${index + 1}. ${entry}`);
    });

    return result;
  }
}

// Usage Example
async function runLoanProcessing() {
  const loanSystem = new LoanProcessingGraph();

  const application = {
    applicationId: 'LOAN-2025-001',
    customerName: 'Ahmed Al Maktoum',
    customerId: 'CUST-12345',
    loanAmount: 500000,
    loanPurpose: 'Home Purchase',
    monthlyIncome: 35000,
    existingDebts: 5000,
    employmentYears: 8,
    propertyValue: 750000
  };

  const result = await loanSystem.process(application);

  return result;
}
```

---

## 3. CrewAI Multi-Agent System

### Role-Based Agent Collaboration

```typescript
/**
 * CrewAI-style Multi-Agent System for Document Verification
 */

interface AgentRole {
  name: string;
  goal: string;
  backstory: string;
  tools: string[];
}

interface Task {
  id: string;
  description: string;
  agent: string;
  expectedOutput: string;
  context?: string[];
}

/**
 * Document Verification Crew
 */
class DocumentVerificationCrew {
  private agents: Map<string, AgentRole> = new Map();
  private tasks: Task[] = [];

  constructor() {
    this.defineAgents();
    this.defineTasks();
  }

  private defineAgents() {
    // Agent 1: Document Extractor
    this.agents.set('extractor', {
      name: 'Document Extractor',
      goal: 'Extract all relevant information from KYC documents',
      backstory: 'Expert in OCR and document parsing with 10+ years experience in banking KYC',
      tools: ['OCR', 'PDF Parser', 'Image Analysis']
    });

    // Agent 2: Verification Specialist
    this.agents.set('verifier', {
      name: 'Verification Specialist',
      goal: 'Verify authenticity and accuracy of extracted data',
      backstory: 'Compliance expert specializing in UAE banking regulations and fraud detection',
      tools: ['Emirates ID Verification API', 'Database Lookup', 'Pattern Matching']
    });

    // Agent 3: Compliance Checker
    this.agents.set('compliance', {
      name: 'Compliance Checker',
      goal: 'Ensure all regulatory requirements are met',
      backstory: 'UAE Central Bank compliance specialist with expertise in AML/KYC regulations',
      tools: ['Regulatory Database', 'Sanction Lists', 'PEP Screening']
    });

    // Agent 4: Report Generator
    this.agents.set('reporter', {
      name: 'Report Generator',
      goal: 'Create comprehensive verification report',
      backstory: 'Documentation specialist creating clear, actionable reports for decision makers',
      tools: ['Report Templates', 'PDF Generator', 'Email Service']
    });
  }

  private defineTasks() {
    this.tasks = [
      {
        id: 'extract',
        description: 'Extract customer information from Emirates ID, passport, and salary certificate',
        agent: 'extractor',
        expectedOutput: 'Structured JSON with all extracted fields'
      },
      {
        id: 'verify',
        description: 'Verify extracted information against official databases',
        agent: 'verifier',
        expectedOutput: 'Verification status with confidence scores',
        context: ['extract']
      },
      {
        id: 'check_compliance',
        description: 'Check compliance with UAE banking regulations, AML, and KYC requirements',
        agent: 'compliance',
        expectedOutput: 'Compliance checklist with pass/fail status',
        context: ['extract', 'verify']
      },
      {
        id: 'generate_report',
        description: 'Generate final verification report for account opening',
        agent: 'reporter',
        expectedOutput: 'PDF report with recommendations',
        context: ['extract', 'verify', 'check_compliance']
      }
    ];
  }

  async execute(documents: any[]): Promise<any> {
    console.log(`\n👥 Document Verification Crew - Starting...`);
    console.log(`📄 Documents to process: ${documents.length}\n`);

    const results: Map<string, any> = new Map();

    for (const task of this.tasks) {
      const agent = this.agents.get(task.agent);
      console.log(`\n🤖 ${agent?.name} - ${task.description}`);

      // Get context from previous tasks
      const context = task.context?.map(taskId => results.get(taskId)) || [];

      // Simulate agent execution
      const result = await this.executeAgent(task, documents, context);
      results.set(task.id, result);

      console.log(`✅ Task completed: ${task.expectedOutput}`);
    }

    console.log(`\n✅ All agents completed their tasks!`);

    return {
      extractedData: results.get('extract'),
      verificationStatus: results.get('verify'),
      complianceCheck: results.get('check_compliance'),
      finalReport: results.get('generate_report')
    };
  }

  private async executeAgent(task: Task, documents: any[], context: any[]): Promise<any> {
    const agent = this.agents.get(task.agent);
    const model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0,
      openAIApiKey: process.env.OPENAI_API_KEY
    });

    const prompt = `You are ${agent?.name} at Emirates NBD.

Goal: ${agent?.goal}
Background: ${agent?.backstory}
Available Tools: ${agent?.tools.join(', ')}

Task: ${task.description}
Expected Output: ${task.expectedOutput}

Documents: ${JSON.stringify(documents)}
Previous Results: ${JSON.stringify(context)}

Complete the task and provide structured output.`;

    const response = await model.call(prompt);

    return {
      agent: agent?.name,
      output: response.content,
      timestamp: new Date()
    };
  }
}

// Usage Example
async function runDocumentVerification() {
  const crew = new DocumentVerificationCrew();

  const documents = [
    {
      type: 'EmiratesID',
      idNumber: '784-1234-5678901-2',
      name: 'Ahmed Al Maktoum',
      dateOfBirth: '1985-03-15',
      expiryDate: '2027-03-14'
    },
    {
      type: 'Passport',
      passportNumber: 'A12345678',
      nationality: 'UAE',
      issueDate: '2020-01-10',
      expiryDate: '2030-01-09'
    },
    {
      type: 'SalaryCertificate',
      employer: 'Emirates Airlines',
      monthlySalary: 35000,
      employmentDate: '2015-06-01'
    }
  ];

  const result = await crew.execute(documents);
  
  console.log('\n📊 Verification Results:');
  console.log(JSON.stringify(result, null, 2));
}
```

---

## 4. Hierarchical Multi-Agent System

### Manager-Worker Pattern

```typescript
/**
 * Hierarchical Multi-Agent System
 * Manager delegates tasks to specialized workers
 */

interface WorkerAgent {
  id: string;
  name: string;
  specialty: string;
  availability: boolean;
  currentLoad: number;
}

/**
 * Manager Agent - Orchestrates workers
 */
class ManagerAgent {
  private workers: WorkerAgent[] = [];
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({
      modelName: 'gpt-4',
      temperature: 0.3,
      openAIApiKey: process.env.OPENAI_API_KEY
    });

    this.initializeWorkers();
  }

  private initializeWorkers() {
    this.workers = [
      {
        id: 'worker-1',
        name: 'Fraud Detection Specialist',
        specialty: 'fraud_detection',
        availability: true,
        currentLoad: 0
      },
      {
        id: 'worker-2',
        name: 'Transaction Analyst',
        specialty: 'transaction_analysis',
        availability: true,
        currentLoad: 0
      },
      {
        id: 'worker-3',
        name: 'Customer Profiler',
        specialty: 'customer_profiling',
        availability: true,
        currentLoad: 0
      },
      {
        id: 'worker-4',
        name: 'Risk Assessor',
        specialty: 'risk_assessment',
        availability: true,
        currentLoad: 0
      }
    ];
  }

  async delegateTask(task: string, taskType: string): Promise<any> {
    console.log(`\n[Manager] Received task: ${task}`);
    console.log(`[Manager] Task type: ${taskType}`);

    // Find appropriate worker
    const worker = this.selectWorker(taskType);

    if (!worker) {
      console.log(`[Manager] No available worker for ${taskType}`);
      return { error: 'No available worker' };
    }

    console.log(`[Manager] Delegating to: ${worker.name}`);

    // Mark worker as busy
    worker.availability = false;
    worker.currentLoad++;

    // Execute task with worker
    const result = await this.executeWorkerTask(worker, task);

    // Free up worker
    worker.availability = true;
    worker.currentLoad--;

    console.log(`[Manager] Task completed by ${worker.name}`);

    return result;
  }

  private selectWorker(taskType: string): WorkerAgent | null {
    // Load balancing - select available worker with matching specialty and lowest load
    const availableWorkers = this.workers.filter(
      w => w.specialty === taskType && w.availability
    );

    if (availableWorkers.length === 0) {
      return null;
    }

    // Return worker with lowest current load
    return availableWorkers.reduce((prev, current) =>
      prev.currentLoad < current.currentLoad ? prev : current
    );
  }

  private async executeWorkerTask(worker: WorkerAgent, task: string): Promise<any> {
    const prompt = `You are ${worker.name}, a ${worker.specialty} specialist at Emirates NBD.

Task: ${task}

Provide detailed analysis based on your expertise. Include:
1. Key findings
2. Risk indicators
3. Recommendations
4. Confidence score (0-100)

Return structured JSON.`;

    const response = await this.model.call(prompt);

    return {
      worker: worker.name,
      specialty: worker.specialty,
      result: response.content,
      timestamp: new Date()
    };
  }

  async processBatch(tasks: Array<{ task: string; type: string }>): Promise<any[]> {
    console.log(`\n[Manager] Processing batch of ${tasks.length} tasks...`);

    const results = await Promise.all(
      tasks.map(t => this.delegateTask(t.task, t.type))
    );

    console.log(`[Manager] Batch processing complete!`);

    return results;
  }

  getWorkerStatus(): WorkerAgent[] {
    return this.workers.map(w => ({
      ...w,
      status: w.availability ? 'Available' : 'Busy'
    }));
  }
}

// Usage Example
async function runHierarchicalSystem() {
  const manager = new ManagerAgent();

  // Single task
  const result1 = await manager.delegateTask(
    'Analyze transaction TXN-12345 for fraud indicators',
    'fraud_detection'
  );

  console.log('\n📊 Result:', result1);

  // Batch processing
  const tasks = [
    { task: 'Analyze customer spending patterns for CUST-001', type: 'customer_profiling' },
    { task: 'Assess risk for new loan application LOAN-456', type: 'risk_assessment' },
    { task: 'Detect anomalies in transaction batch', type: 'fraud_detection' },
    { task: 'Profile high-value transactions', type: 'transaction_analysis' }
  ];

  const batchResults = await manager.processBatch(tasks);

  console.log('\n📊 Worker Status:');
  console.log(manager.getWorkerStatus());
}
```

---

## Key Takeaways

### Multi-Agent Architecture Patterns:

1. **Sequential (Chain)**: Agents execute one after another
   - Best for: Linear workflows, document processing
   - Example: Research → Analysis → Report

2. **Parallel (Crew)**: Multiple agents work simultaneously
   - Best for: Independent subtasks, speed optimization
   - Example: Multiple document verifiers

3. **Hierarchical (Manager-Worker)**: Manager delegates to specialized workers
   - Best for: Complex tasks, load balancing
   - Example: Task routing, resource allocation

4. **Graph-Based (LangGraph)**: State-driven workflows with conditional paths
   - Best for: Complex decision trees, adaptive workflows
   - Example: Loan processing with multiple approval paths

### When to Use Multi-Agent Systems:

✅ **Use Multi-Agent When:**
- Task requires multiple areas of expertise
- Parallel processing can improve speed
- Need modular, maintainable architecture
- Different agents need different models/temperatures
- Complex workflows with decision points

❌ **Avoid Multi-Agent When:**
- Simple, single-step tasks
- Latency is critical (each agent adds overhead)
- Budget constrained (multiple LLM calls)
- Task doesn't benefit from specialization

### Best Practices:

1. ✅ Define clear roles and responsibilities
2. ✅ Implement proper error handling and fallbacks
3. ✅ Monitor agent performance and costs
4. ✅ Use state management for complex workflows
5. ✅ Implement timeouts and circuit breakers
6. ✅ Log all agent interactions for debugging
7. ✅ Test each agent independently before integration
8. ✅ **Assign specialized tools to each agent**
9. ✅ **Validate tool outputs before passing to next agent**
10. ✅ **Cache tool results to avoid redundant API calls**

---

## Tools Integration in Multi-Agent Systems

### Agent-Specific Tools

Each agent type should have tools specific to its role:

```typescript
// Fraud Detection Agent Tools
const fraudAgentTools = [
  checkTransactionPatternTool,
  calculateFraudScoreTool,
  getTransactionHistoryTool,
  checkMerchantReputationTool
];

// Credit Assessment Agent Tools
const creditAgentTools = [
  getCreditScoreTool,
  checkCreditBureauTool,
  calculateDTITool,
  verifySalaryTool
];

// Compliance Agent Tools
const complianceAgentTools = [
  checkSanctionsListTool,
  verifyPEPStatusTool,
  performAMLCheckTool,
  verifyEmiratesIDTool
];
```

### Tool Calling Example in Sequential Agents

```typescript
class ResearchAgentWithTools {
  private tools = [
    new DynamicStructuredTool({
      name: 'search_banking_regulations',
      description: 'Search UAE Central Bank regulations',
      schema: z.object({
        query: z.string(),
        category: z.enum(['lending', 'kyc', 'aml', 'reporting'])
      }),
      func: async ({ query, category }) => {
        const results = await searchRegulations(query, category);
        return JSON.stringify(results);
      }
    }),
    new DynamicStructuredTool({
      name: 'get_market_data',
      description: 'Get current market data for analysis',
      schema: z.object({
        dataType: z.enum(['interest_rates', 'currency', 'property_prices'])
      }),
      func: async ({ dataType }) => {
        const data = await fetchMarketData(dataType);
        return JSON.stringify(data);
      }
    })
  ];

  async execute(input: string, context?: any): Promise<AgentResult> {
    const agent = await createToolCallingAgent({
      llm: this.model,
      tools: this.tools,
      prompt: this.prompt
    });

    const executor = new AgentExecutor({ agent, tools: this.tools });
    const result = await executor.invoke({ input });

    return {
      agentName: this.name,
      output: result.output,
      toolsUsed: result.intermediateSteps?.map(step => step.action.tool) || [],
      nextAgent: 'AnalysisAgent'
    };
  }
}
```

### Tool Sharing Between Agents

```typescript
// Shared tool that multiple agents can use
const customerDataTool = new DynamicStructuredTool({
  name: 'get_customer_data',
  description: 'Retrieve customer information',
  schema: z.object({
    customerId: z.string(),
    agentRole: z.string()  // Track which agent is calling
  }),
  func: async ({ customerId, agentRole }) => {
    console.log(`[${agentRole}] Accessing customer ${customerId}`);
    
    // Apply role-based data filtering
    const data = await fetchCustomerData(customerId);
    const filtered = filterDataByRole(data, agentRole);
    
    return JSON.stringify(filtered);
  }
});

// Multiple agents use the same tool
const fraudAgent = createAgent({
  tools: [customerDataTool, fraudScoreTool],
  systemMessage: 'You are a fraud detection specialist'
});

const creditAgent = createAgent({
  tools: [customerDataTool, creditBureauTool],
  systemMessage: 'You are a credit assessment specialist'
});
```

---

## 5. Real-World Example: QA Automation Multi-Agent System

### Complete 6-Agent Implementation

```typescript
import { ChatOpenAI } from '@langchain/openai';
import { StateGraph } from '@langchain/langgraph';
import { DynamicStructuredTool } from 'langchain/tools';
import { z } from 'zod';
import Redis from 'ioredis';
import { Pinecone } from '@pinecone-database/pinecone';
import { Client } from 'pg';
import AWS from 'aws-sdk';
import puppeteer from 'puppeteer';

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
  sops?: {
    sopId: string;
    content: string;
    relevanceScore: number;
  }[];
  testPlan?: {
    testCases: TestCase[];
    coverage: number;
  };
  playwrightScripts?: string[];
  testResults?: {
    passed: number;
    failed: number;
    screenshots: string[];
    logs: string[];
  };
  artifacts?: {
    s3Urls: string[];
    signedUrls: string[];
  };
  pdfReport?: {
    url: string;
    generatedAt: string;
  };
  manualReview?: boolean;
  retryCount?: number;
  error?: string;
}

interface TestCase {
  id: string;
  title: string;
  steps: string[];
  expectedResult: string;
  priority: 'high' | 'medium' | 'low';
}

/**
 * Agent 1: JIRA Reader
 * Fetches ticket details and parses requirements
 */
class JiraReaderAgent {
  private model: ChatOpenAI;
  private tools: DynamicStructuredTool[];

  constructor() {
    this.model = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });
    this.tools = [this.createFetchJiraTicketTool()];
  }

  createFetchJiraTicketTool() {
    return new DynamicStructuredTool({
      name: 'fetch_jira_ticket',
      description: 'Fetches JIRA ticket details including description and acceptance criteria',
      schema: z.object({
        ticketId: z.string().describe('JIRA ticket ID (e.g., QA-1234)'),
      }),
      func: async ({ ticketId }) => {
        const response = await fetch(
          `https://jira.company.com/rest/api/2/issue/${ticketId}`,
          {
            headers: {
              Authorization: `Bearer ${process.env.JIRA_TOKEN}`,
              'Content-Type': 'application/json',
            },
          }
        );
        const data = await response.json();
        return JSON.stringify({
          summary: data.fields.summary,
          description: data.fields.description,
          acceptanceCriteria: data.fields.customfield_10100,
        });
      },
    });
  }

  async execute(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
    console.log(`[JIRA Reader] Fetching ticket ${state.ticketId}...`);

    const modelWithTools = this.model.bind({ tools: this.tools });
    
    const response = await modelWithTools.invoke([
      {
        role: 'system',
        content: 'You are a JIRA ticket analysis expert. Extract and structure ticket information.',
      },
      {
        role: 'user',
        content: `Fetch and analyze JIRA ticket ${state.ticketId}. Extract:
1. Summary
2. Description
3. Acceptance criteria
4. Key requirements`,
      },
    ]);

    const ticketData = JSON.parse(response.content as string);

    return {
      ticketDetails: {
        summary: ticketData.summary,
        description: ticketData.description,
        acceptanceCriteria: ticketData.acceptanceCriteria,
      },
    };
  }
}

/**
 * Agent 2: SOP Analyzer
 * Searches for relevant SOPs using vector similarity
 */
class SopAnalyzerAgent {
  private model: ChatOpenAI;
  private pinecone: Pinecone;

  constructor() {
    this.model = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });
    this.pinecone = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY!,
    });
  }

  async execute(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
    console.log('[SOP Analyzer] Searching for relevant SOPs...');

    const index = this.pinecone.index('sop-embeddings');

    // Generate embedding for ticket description
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
          input: state.ticketDetails?.description,
        }),
      }
    );
    const embeddingData = await embeddingResponse.json();
    const embedding = embeddingData.data[0].embedding;

    // Search for similar SOPs
    const searchResults = await index.query({
      vector: embedding,
      topK: 3,
      includeMetadata: true,
    });

    const sops = searchResults.matches?.map((match) => ({
      sopId: match.id,
      content: match.metadata?.content as string,
      relevanceScore: match.score || 0,
    }));

    return { sops };
  }
}

/**
 * Agent 3: Test Generator
 * Creates test plan and generates Playwright scripts
 */
class TestGeneratorAgent {
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });
  }

  async execute(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
    console.log('[Test Generator] Creating test plan...');

    // Generate test plan
    const testPlanPrompt = `Based on the following JIRA ticket and SOPs, create a comprehensive test plan:

Ticket: ${state.ticketDetails?.summary}
Description: ${state.ticketDetails?.description}
Acceptance Criteria: ${state.ticketDetails?.acceptanceCriteria.join('\n')}

SOPs:
${state.sops?.map((sop) => `- ${sop.sopId}: ${sop.content}`).join('\n')}

Create test cases with:
1. Test case ID
2. Title
3. Steps
4. Expected result
5. Priority (high/medium/low)

Format as JSON array.`;

    const testPlanResponse = await this.model.invoke(testPlanPrompt);
    const testCases = JSON.parse(testPlanResponse.content as string);

    // Generate Playwright scripts
    const playwrightScripts = await Promise.all(
      testCases.map((testCase: TestCase) =>
        this.generatePlaywrightScript(testCase)
      )
    );

    return {
      testPlan: {
        testCases,
        coverage: this.calculateCoverage(testCases, state.ticketDetails!),
      },
      playwrightScripts,
    };
  }

  async generatePlaywrightScript(testCase: TestCase): Promise<string> {
    const scriptPrompt = `Generate a Playwright test script for:

Test Case: ${testCase.title}
Steps: ${testCase.steps.join('\n')}
Expected Result: ${testCase.expectedResult}

Include:
1. Page navigation
2. Element interactions
3. Assertions
4. Screenshot capture on failure
5. Error handling`;

    const response = await this.model.invoke(scriptPrompt);
    return response.content as string;
  }

  calculateCoverage(
    testCases: TestCase[],
    ticketDetails: QAAutomationState['ticketDetails']
  ): number {
    const totalCriteria = ticketDetails?.acceptanceCriteria.length || 0;
    const coveredCriteria = testCases.filter((tc) => tc.priority === 'high').length;
    return (coveredCriteria / totalCriteria) * 100;
  }
}

/**
 * Agent 4: Test Executor
 * Runs Playwright tests and captures screenshots/logs
 */
class TestExecutorAgent {
  async execute(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
    console.log('[Test Executor] Running Playwright tests...');

    const results = {
      passed: 0,
      failed: 0,
      screenshots: [] as string[],
      logs: [] as string[],
    };

    for (const script of state.playwrightScripts || []) {
      try {
        const result = await this.executePlaywrightTest(script);
        if (result.success) {
          results.passed++;
        } else {
          results.failed++;
        }
        results.screenshots.push(...result.screenshots);
        results.logs.push(result.log);
      } catch (error) {
        results.failed++;
        results.logs.push(`Error: ${error}`);
      }
    }

    return { testResults: results };
  }

  async executePlaywrightTest(script: string): Promise<{
    success: boolean;
    screenshots: string[];
    log: string;
  }> {
    // Execute Playwright script
    const { chromium } = require('playwright');
    const browser = await chromium.launch();
    const page = await browser.newPage();

    const screenshots: string[] = [];
    const logs: string[] = [];

    try {
      // Execute the script (simplified)
      eval(script); // In production, use proper script execution

      // Capture screenshot
      const screenshotBuffer = await page.screenshot();
      const screenshotPath = `/tmp/screenshot_${Date.now()}.png`;
      await require('fs').promises.writeFile(screenshotPath, screenshotBuffer);
      screenshots.push(screenshotPath);

      logs.push('Test passed successfully');
      await browser.close();

      return { success: true, screenshots, log: logs.join('\n') };
    } catch (error) {
      logs.push(`Test failed: ${error}`);
      
      // Capture failure screenshot
      const screenshotBuffer = await page.screenshot();
      const screenshotPath = `/tmp/failure_${Date.now()}.png`;
      await require('fs').promises.writeFile(screenshotPath, screenshotBuffer);
      screenshots.push(screenshotPath);

      await browser.close();
      return { success: false, screenshots, log: logs.join('\n') };
    }
  }
}

/**
 * Agent 5: Artifact Manager
 * Uploads screenshots to S3 and generates signed URLs
 */
class ArtifactManagerAgent {
  private s3: AWS.S3;

  constructor() {
    this.s3 = new AWS.S3({
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
      region: process.env.AWS_REGION,
    });
  }

  async execute(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
    console.log('[Artifact Manager] Uploading artifacts to S3...');

    const s3Urls: string[] = [];
    const signedUrls: string[] = [];

    // Upload screenshots
    for (const screenshotPath of state.testResults?.screenshots || []) {
      const fileContent = await require('fs').promises.readFile(screenshotPath);
      const key = `qa-automation/${state.ticketId}/${Date.now()}_${screenshotPath.split('/').pop()}`;

      await this.s3
        .putObject({
          Bucket: process.env.S3_BUCKET!,
          Key: key,
          Body: fileContent,
          ContentType: 'image/png',
        })
        .promise();

      const s3Url = `https://${process.env.S3_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
      s3Urls.push(s3Url);

      // Generate signed URL (valid for 7 days)
      const signedUrl = this.s3.getSignedUrl('getObject', {
        Bucket: process.env.S3_BUCKET!,
        Key: key,
        Expires: 604800, // 7 days
      });
      signedUrls.push(signedUrl);
    }

    return {
      artifacts: { s3Urls, signedUrls },
    };
  }
}

/**
 * Agent 6: Report Generator
 * Creates PDF report with screenshots and sends notifications
 */
class ReportGeneratorAgent {
  private model: ChatOpenAI;

  constructor() {
    this.model = new ChatOpenAI({ model: 'gpt-4', temperature: 0 });
  }

  async execute(state: QAAutomationState): Promise<Partial<QAAutomationState>> {
    console.log('[Report Generator] Creating PDF report...');

    // Generate report content
    const reportContent = await this.generateReportContent(state);

    // Create PDF using Puppeteer
    const pdfBuffer = await this.createPdf(reportContent, state);

    // Upload PDF to S3
    const s3 = new AWS.S3();
    const pdfKey = `qa-automation/${state.ticketId}/report_${Date.now()}.pdf`;
    await s3
      .putObject({
        Bucket: process.env.S3_BUCKET!,
        Key: pdfKey,
        Body: pdfBuffer,
        ContentType: 'application/pdf',
      })
      .promise();

    const pdfUrl = `https://${process.env.S3_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${pdfKey}`;

    // Send notification
    await this.sendNotification(state, pdfUrl);

    // Update JIRA ticket
    await this.updateJiraTicket(state, pdfUrl);

    return {
      pdfReport: {
        url: pdfUrl,
        generatedAt: new Date().toISOString(),
      },
    };
  }

  async generateReportContent(state: QAAutomationState): Promise<string> {
    const prompt = `Generate a comprehensive QA test report:

Ticket: ${state.ticketId} - ${state.ticketDetails?.summary}
Test Results: ${state.testResults?.passed} passed, ${state.testResults?.failed} failed
Coverage: ${state.testPlan?.coverage}%
Screenshots: ${state.artifacts?.signedUrls.length} captured

Include:
1. Executive summary
2. Test coverage analysis
3. Pass/fail breakdown
4. Screenshots with descriptions
5. Recommendations`;

    const response = await this.model.invoke(prompt);
    return response.content as string;
  }

  async createPdf(
    content: string,
    state: QAAutomationState
  ): Promise<Buffer> {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();

    const html = `
<!DOCTYPE html>
<html>
<head>
  <title>QA Test Report - ${state.ticketId}</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 40px; }
    h1 { color: #333; }
    .summary { background: #f5f5f5; padding: 20px; margin: 20px 0; }
    .screenshot { margin: 20px 0; }
    .screenshot img { max-width: 800px; border: 1px solid #ddd; }
  </style>
</head>
<body>
  <h1>QA Test Report</h1>
  <div class="summary">
    <h2>Summary</h2>
    <p><strong>Ticket:</strong> ${state.ticketId}</p>
    <p><strong>Summary:</strong> ${state.ticketDetails?.summary}</p>
    <p><strong>Passed:</strong> ${state.testResults?.passed}</p>
    <p><strong>Failed:</strong> ${state.testResults?.failed}</p>
    <p><strong>Coverage:</strong> ${state.testPlan?.coverage}%</p>
  </div>
  
  <h2>Test Results</h2>
  ${content}
  
  <h2>Screenshots</h2>
  ${state.artifacts?.signedUrls
    .map(
      (url, idx) => `
    <div class="screenshot">
      <h3>Screenshot ${idx + 1}</h3>
      <img src="${url}" alt="Test screenshot ${idx + 1}" />
    </div>
  `
    )
    .join('')}
  
  <h2>Logs</h2>
  <pre>${state.testResults?.logs.join('\n\n')}</pre>
</body>
</html>`;

    await page.setContent(html);
    const pdfBuffer = await page.pdf({ format: 'A4' });
    await browser.close();

    return pdfBuffer;
  }

  async sendNotification(state: QAAutomationState, pdfUrl: string) {
    // Send email via SendGrid
    const sgMail = require('@sendgrid/mail');
    sgMail.setApiKey(process.env.SENDGRID_API_KEY);

    const msg = {
      to: 'qa-team@company.com',
      from: 'automation@company.com',
      subject: `QA Test Report - ${state.ticketId}`,
      html: `
<h2>Test Execution Complete</h2>
<p><strong>Ticket:</strong> ${state.ticketId}</p>
<p><strong>Summary:</strong> ${state.ticketDetails?.summary}</p>
<p><strong>Passed:</strong> ${state.testResults?.passed}</p>
<p><strong>Failed:</strong> ${state.testResults?.failed}</p>
<p><a href="${pdfUrl}">Download Full Report</a></p>
      `,
    };

    await sgMail.send(msg);
  }

  async updateJiraTicket(state: QAAutomationState, pdfUrl: string) {
    await fetch(
      `https://jira.company.com/rest/api/2/issue/${state.ticketId}/comment`,
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${process.env.JIRA_TOKEN}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          body: `Automated testing completed.
Passed: ${state.testResults?.passed}
Failed: ${state.testResults?.failed}
Report: ${pdfUrl}`,
        }),
      }
    );
  }
}

/**
 * LangGraph Workflow with Conditional Routing
 */
function createQaAutomationWorkflow() {
  const workflow = new StateGraph<QAAutomationState>({
    channels: {
      ticketId: null,
      ticketDetails: null,
      sops: null,
      testPlan: null,
      playwrightScripts: null,
      testResults: null,
      artifacts: null,
      pdfReport: null,
      manualReview: null,
      retryCount: null,
      error: null,
    },
  });

  // Initialize agents
  const jiraReader = new JiraReaderAgent();
  const sopAnalyzer = new SopAnalyzerAgent();
  const testGenerator = new TestGeneratorAgent();
  const testExecutor = new TestExecutorAgent();
  const artifactManager = new ArtifactManagerAgent();
  const reportGenerator = new ReportGeneratorAgent();

  // Add nodes
  workflow.addNode('jiraReader', async (state) => {
    try {
      return await jiraReader.execute(state);
    } catch (error) {
      return { error: `JIRA Reader failed: ${error}` };
    }
  });

  workflow.addNode('sopAnalyzer', async (state) => {
    try {
      return await sopAnalyzer.execute(state);
    } catch (error) {
      return { error: `SOP Analyzer failed: ${error}` };
    }
  });

  workflow.addNode('testGenerator', async (state) => {
    try {
      return await testGenerator.execute(state);
    } catch (error) {
      return { error: `Test Generator failed: ${error}` };
    }
  });

  workflow.addNode('manualReview', async (state) => {
    console.log('[Manual Review] Waiting for human approval...');
    // In production, this would trigger a UI for review
    return { manualReview: true };
  });

  workflow.addNode('testExecutor', async (state) => {
    try {
      return await testExecutor.execute(state);
    } catch (error) {
      return { error: `Test Executor failed: ${error}` };
    }
  });

  workflow.addNode('artifactManager', async (state) => {
    try {
      return await artifactManager.execute(state);
    } catch (error) {
      return { error: `Artifact Manager failed: ${error}` };
    }
  });

  workflow.addNode('reportGenerator', async (state) => {
    try {
      return await reportGenerator.execute(state);
    } catch (error) {
      return { error: `Report Generator failed: ${error}` };
    }
  });

  // Define edges with conditional routing
  workflow.setEntryPoint('jiraReader');
  workflow.addEdge('jiraReader', 'sopAnalyzer');
  workflow.addEdge('sopAnalyzer', 'testGenerator');

  // Conditional: Manual review for complex tickets
  workflow.addConditionalEdges('testGenerator', (state) => {
    const complexity = state.testPlan?.testCases.length || 0;
    return complexity > 10 ? 'manualReview' : 'testExecutor';
  });

  workflow.addEdge('manualReview', 'testExecutor');
  workflow.addEdge('testExecutor', 'artifactManager');
  workflow.addEdge('artifactManager', 'reportGenerator');

  // Conditional: Retry on failure
  workflow.addConditionalEdges('reportGenerator', (state) => {
    const failureRate =
      (state.testResults?.failed || 0) /
      ((state.testResults?.passed || 0) + (state.testResults?.failed || 0));

    if (failureRate > 0.5 && (state.retryCount || 0) < 3) {
      console.log('[Workflow] High failure rate, retrying...');
      return 'testGenerator'; // Regenerate tests
    }

    return '__end__';
  });

  return workflow.compile();
}

/**
 * Main execution
 */
async function main() {
  const app = createQaAutomationWorkflow();

  const result = await app.invoke({
    ticketId: 'QA-1234',
    retryCount: 0,
  });

  console.log('\n=== Final Result ===');
  console.log(`Ticket: ${result.ticketId}`);
  console.log(`Test Coverage: ${result.testPlan?.coverage}%`);
  console.log(`Passed: ${result.testResults?.passed}`);
  console.log(`Failed: ${result.testResults?.failed}`);
  console.log(`PDF Report: ${result.pdfReport?.url}`);
}

// main().catch(console.error);
```

### Memory Management for QA System

```typescript
import Redis from 'ioredis';
import { Pinecone } from '@pinecone-database/pinecone';
import { Pool } from 'pg';

/**
 * 3-Tier Memory Architecture
 */
class QaMemoryManager {
  private redis: Redis;
  private pinecone: Pinecone;
  private postgres: Pool;

  constructor() {
    // Short-term memory (24h TTL)
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT || '6379'),
    });

    // Long-term semantic memory
    this.pinecone = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY!,
    });

    // Entity memory
    this.postgres = new Pool({
      host: process.env.POSTGRES_HOST,
      database: process.env.POSTGRES_DB,
      user: process.env.POSTGRES_USER,
      password: process.env.POSTGRES_PASSWORD,
    });
  }

  // Short-term: Recent test runs
  async saveRecentTestRun(ticketId: string, results: any) {
    await this.redis.setex(
      `test_run:${ticketId}`,
      86400, // 24 hours
      JSON.stringify(results)
    );
  }

  async getRecentTestRun(ticketId: string) {
    const data = await this.redis.get(`test_run:${ticketId}`);
    return data ? JSON.parse(data) : null;
  }

  // Long-term: SOP embeddings and similar ticket search
  async storeSopEmbedding(sopId: string, content: string) {
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
          input: content,
        }),
      }
    );
    const embeddingData = await embeddingResponse.json();
    const embedding = embeddingData.data[0].embedding;

    const index = this.pinecone.index('sop-embeddings');
    await index.upsert([
      {
        id: sopId,
        values: embedding,
        metadata: { content, createdAt: new Date().toISOString() },
      },
    ]);
  }

  async findSimilarTickets(ticketDescription: string, topK: number = 5) {
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
          input: ticketDescription,
        }),
      }
    );
    const embeddingData = await embeddingResponse.json();
    const embedding = embeddingData.data[0].embedding;

    const index = this.pinecone.index('ticket-history');
    const results = await index.query({
      vector: embedding,
      topK,
      includeMetadata: true,
    });

    return results.matches?.map((match) => ({
      ticketId: match.id,
      similarity: match.score,
      metadata: match.metadata,
    }));
  }

  // Entity: Team preferences, project SOPs, test histories
  async saveTeamPreference(teamId: string, preferences: any) {
    await this.postgres.query(
      'INSERT INTO team_preferences (team_id, preferences, updated_at) VALUES ($1, $2, NOW()) ON CONFLICT (team_id) DO UPDATE SET preferences = $2, updated_at = NOW()',
      [teamId, JSON.stringify(preferences)]
    );
  }

  async getTestHistory(ticketId: string) {
    const result = await this.postgres.query(
      'SELECT * FROM test_history WHERE ticket_id = $1 ORDER BY executed_at DESC LIMIT 10',
      [ticketId]
    );
    return result.rows;
  }

  async saveTestHistory(ticketId: string, testResults: any) {
    await this.postgres.query(
      'INSERT INTO test_history (ticket_id, results, executed_at) VALUES ($1, $2, NOW())',
      [ticketId, JSON.stringify(testResults)]
    );
  }
}
```

### ROI Calculation

```typescript
/**
 * QA Automation ROI Calculator
 */
class QaAutomationRoi {
  calculateSavings() {
    const manualTestingTime = 4.5; // hours per ticket
    const automatedTestingTime = 0.15; // hours (9 minutes)
    const ticketsPerWeek = 20;
    const qaEngineerHourlyRate = 40; // USD

    const timeSavedPerTicket = manualTestingTime - automatedTestingTime; // 4.35 hours
    const timeSavedPerWeek = timeSavedPerTicket * ticketsPerWeek; // 87 hours
    const costSavingsPerWeek = timeSavedPerWeek * qaEngineerHourlyRate; // $3,480

    const yearlySavings = costSavingsPerWeek * 52; // $180,960

    // Account for initial setup and maintenance
    const setupCost = 20000; // USD
    const maintenanceCostPerYear = 36000; // USD

    const netYearlySavings =
      yearlySavings - setupCost / 3 - maintenanceCostPerYear; // $138,293

    return {
      timeSavedPerTicket,
      timeSavedPerWeek,
      costSavingsPerWeek,
      yearlySavings,
      netYearlySavings,
      roi: ((netYearlySavings / (setupCost + maintenanceCostPerYear)) * 100).toFixed(
        2
      ),
    };
  }
}

const roi = new QaAutomationRoi();
console.log('QA Automation ROI:', roi.calculateSavings());
// Output:
// {
//   timeSavedPerTicket: 4.35,
//   timeSavedPerWeek: 87,
//   costSavingsPerWeek: 3480,
//   yearlySavings: 180960,
//   netYearlySavings: 138293.33,
//   roi: '247.13%'
// }
```

---

**Banking Applications:**
- Loan processing (credit, risk, approval agents **with specialized tools**)
- KYC verification (extraction, verification, compliance agents **with document tools**)
- Fraud detection (analysis, investigation, reporting agents **with pattern detection tools**)
- Customer service (routing, specialist, escalation agents **with CRM tools**)
- Investment advisory (research, analysis, recommendation agents **with market data tools**)

**QA Automation Applications:**
- Automated test generation from JIRA tickets
- SOP compliance verification
- Regression testing with Playwright
- Artifact management (screenshots, logs)
- PDF report generation with stakeholder notifications

---

**Related Files:**
- `QA_Automation_Scenario.md` - Complete QA system architecture
- `Tools_in_Agent_Systems.md` - Comprehensive tools guide
- `Architecture_Diagrams.md` - Visual multi-agent patterns
- `LangGraph_Basics.md` - State management fundamentals



*Next: Q52 - Agent Communication Patterns*
