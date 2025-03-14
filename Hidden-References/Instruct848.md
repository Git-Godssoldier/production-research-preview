# Reasoning Integration Master Plan

## Requirements Summary

This implementation plan outlines the process for integrating our reasoning pattern with a lead orchestrator using OpenAI's Responses API and AI SDK agents. The key requirements include:

1. **Maintain UI/UX:** Preserve the current UI/UX with only text changes
2. **Backend Overhaul:** Restructure the backend while preserving file structure
3. **OpenAI Model Migration:** Move to OpenAI model family with built-in tools
4. **Search Integration:** Use Responses API web search as primary tool, integrated with Jina, Serper, Perplexity, and Firecrawl
5. **Financial Tools:** Incorporate financial tools for agentic workflows
6. **UI Components:** Integrate generative UI finance components
7. **Reasoning Pattern:** Implement structured reasoning with a lead orchestrator

## Implementation Plan

### Phase 1: Model System and Tool Registry (Foundation)

#### 1.1 Update Model System

**File:** `lib/models.ts`
- Replace Anthropic models with OpenAI models
- Implement feature flagging for model capabilities
- Add support for reasoning controls with object-based format
- Create proper typing for model features

```typescript
// Example implementation
import { openai, OpenAILanguageModel } from "@ai-sdk/openai";
import { customProvider } from "ai";

interface ModelCapabilities {
  webSearch: boolean;
  reasoning: boolean;
  responsesApi: boolean;
  parallelTools: boolean;
  financialTools: boolean;
}

interface Model {
  id: string;
  name: string;
  description: string;
  capabilities: ModelCapabilities;
}

export const myProvider = customProvider({
  languageModels: {
    "gpt-4o": openai("gpt-4o"),
    "gpt-4o-mini": openai("gpt-4o-mini"),
    "gpt-4-turbo": openai("gpt-4-turbo"),
  },
});

export const models: Array<Model> = [
  {
    id: "gpt-4o",
    name: "GPT-4o",
    description: "OpenAI's most advanced model with multimodal capabilities.",
    capabilities: {
      webSearch: true,
      reasoning: true,
      responsesApi: true,
      parallelTools: true,
      financialTools: true,
    },
  },
  // Add other models...
];
```

#### 1.2 Create Tool Registry System

**Files to create:**
- `lib/tools/registry.ts`: Central tool registry
- `lib/tools/types.ts`: Tool type definitions
- `lib/tools/validation.ts`: Tool validation using Zod

```typescript
// lib/tools/types.ts example
import { z } from "zod";

export interface Tool<TParams = any, TResult = any> {
  name: string;
  description: string;
  parameters: z.ZodType<TParams>;
  execute: (params: TParams) => Promise<TResult>;
  metadata?: {
    source?: string;
    category?: string;
  };
}

// lib/tools/registry.ts example
import { Tool } from "./types";

class ToolRegistry {
  private tools: Map<string, Tool> = new Map();

  register<TParams, TResult>(tool: Tool<TParams, TResult>): void {
    this.tools.set(tool.name, tool);
  }

  get(name: string): Tool | undefined {
    return this.tools.get(name);
  }

  getAll(): Tool[] {
    return Array.from(this.tools.values());
  }
}

export const toolRegistry = new ToolRegistry();
```

### Phase 2: Search and Financial Integration

#### 2.1 Implement Search Provider Integration

**Files to create:**
- `lib/search/providers/index.ts`: Provider selection logic
- `lib/search/providers/openai.ts`: OpenAI web search integration
- `lib/search/providers/jina.ts`: Jina search APIs
- `lib/search/providers/serper.ts`: Serper API
- `lib/search/providers/perplexity.ts`: Perplexity Sonar
- `lib/search/providers/firecrawl.ts`: Firecrawl API

```typescript
// lib/search/providers/index.ts example
import { searchWithOpenAI } from "./openai";
import { searchWithJina } from "./jina";
import { searchWithSerper } from "./serper";
import { searchWithPerplexity } from "./perplexity";
import { searchWithFirecrawl } from "./firecrawl";

export type SearchProvider = "openai" | "jina" | "serper" | "perplexity" | "firecrawl";

export interface SearchOptions {
  query: string;
  provider?: SearchProvider;
  fallback?: boolean;
  timeoutMs?: number;
}

export interface SearchResult {
  content: string;
  url: string;
  title?: string;
  source: SearchProvider;
}

export async function search(options: SearchOptions): Promise<SearchResult[]> {
  const { provider = "openai", fallback = true } = options;
  
  try {
    switch (provider) {
      case "openai":
        return await searchWithOpenAI(options);
      case "jina":
        return await searchWithJina(options);
      // Other providers...
      default:
        return await searchWithOpenAI(options);
    }
  } catch (error) {
    if (fallback && provider !== "openai") {
      // Try OpenAI as fallback
      return await searchWithOpenAI(options);
    }
    throw error;
  }
}
```

#### 2.2 Implement Financial Tools

**Files to create:**
- `lib/financial/tools/index.ts`: Financial tool integration
- `lib/financial/api/client.ts`: API client for financialdatasets.ai
- `lib/financial/types.ts`: Financial data type definitions

```typescript
// lib/financial/tools/index.ts example
import { z } from "zod";
import { toolRegistry } from "../../tools/registry";
import { getStockPrices, getIncomeStatements, getBalanceSheets, getCashFlowStatements, getFinancialMetrics, searchStocksByFilters } from "../api/client";

// Register getStockPrices tool
toolRegistry.register({
  name: "getStockPrices",
  description: "Use this tool to get stock prices and market cap for a company.",
  parameters: z.object({
    ticker: z.string().describe("The ticker of the company"),
    start_date: z.string().describe("The start date (YYYY-MM-DD)"),
    end_date: z.string().describe("The end date (YYYY-MM-DD)"),
    interval: z.enum(["second", "minute", "day", "week", "month", "year"]).default("day"),
    interval_multiplier: z.number().default(1),
  }),
  execute: async (params) => {
    return await getStockPrices(params);
  },
  metadata: {
    source: "financialdatasets.ai",
    category: "financial",
  },
});

// Register other financial tools...
```

### Phase 3: API and Agent Integration

#### 3.1 Update API Route

**File:** `app/api/chat/route.ts`
- Replace current implementation with Responses API
- Add lead orchestrator agent
- Implement tool registry and calling
- Set up reasoning configuration

```typescript
import { NextRequest } from "next/server";
import { Message, smoothStream, streamText } from "ai";
import { myProvider } from "@/lib/models";
import { toolRegistry } from "@/lib/tools/registry";
import { orchestrateQuery } from "@/lib/agents/orchestrator";
import { prepareTools } from "@/lib/tools/preparation";

export async function POST(request: NextRequest) {
  const {
    messages,
    selectedModelId,
    isReasoningEnabled,
  }: {
    messages: Array<Message>;
    selectedModelId: string;
    isReasoningEnabled: boolean;
  } = await request.json();

  // Get available tools based on model capabilities
  const selectedModel = models.find(model => model.id === selectedModelId);
  const availableTools = prepareTools(selectedModel?.capabilities);

  // Set up reasoning configuration
  const reasoningConfig = isReasoningEnabled ? {
    effort: "comprehensive",
    generate_summary: true,
    budget_tokens: 12000,
  } : undefined;

  // Use orchestrator to manage the flow
  const { prompt, tools } = await orchestrateQuery({
    messages,
    selectedModelId,
    reasoning: reasoningConfig,
  });

  // Use Responses API with streamText
  const stream = streamText({
    system: prompt,
    providerOptions: {
      openai: {
        responses: {
          enabled: true,
          reasoning: reasoningConfig,
          parallel_tool_calls: true,
        },
      },
    },
    model: myProvider.languageModel(selectedModelId),
    experimental_tools: tools,
    experimental_transform: [
      smoothStream({
        chunking: "word",
      }),
    ],
    messages,
  });

  return stream.toDataStreamResponse({
    sendReasoning: isReasoningEnabled,
    getErrorMessage: () => {
      return `An error occurred, please try again!`;
    },
  });
}
```

#### 3.2 Implement Agent Registry and Orchestrator

**Files to create:**
- `lib/agents/registry.ts`: Agent registry with specializations
- `lib/agents/orchestrator.ts`: Lead orchestrator implementation
- `lib/agents/types.ts`: Agent type definitions
- `lib/agents/collaboration.ts`: Collaboration protocols

```typescript
// lib/agents/types.ts example
export type AgentDomain = "General" | "Financial" | "Scientific" | "Legal" | "Medical" | "Technical" | "Creative";
export type AgentRole = "Orchestrator" | "Researcher" | "Analyst" | "Writer" | "Critic" | "Specialist";

export interface Agent {
  name: string;
  domain: AgentDomain;
  role: AgentRole;
  description: string;
  systemPrompt: string;
  defaultModel: string;
  tools: string[]; // Tool names from the registry
}

// lib/agents/registry.ts example
import { Agent } from "./types";

class AgentRegistry {
  private agents: Map<string, Agent> = new Map();

  register(agent: Agent): void {
    this.agents.set(`${agent.domain}-${agent.role}`, agent);
  }

  getByDomainAndRole(domain: AgentDomain, role: AgentRole): Agent | undefined {
    return this.agents.get(`${domain}-${role}`);
  }

  getBestMatch(query: string): Agent {
    // Find the best agent based on semantic similarity
    // For now, return the orchestrator
    return this.getByDomainAndRole("General", "Orchestrator")!;
  }
}

export const agentRegistry = new AgentRegistry();

// Register default agents
agentRegistry.register({
  name: "Lead Orchestrator",
  domain: "General",
  role: "Orchestrator",
  description: "Coordinates the overall workflow and delegates to specialized agents.",
  systemPrompt: "You are a Lead Orchestrator agent responsible for breaking down complex queries...",
  defaultModel: "gpt-4o",
  tools: ["search"],
});

// Register other agents...
```

### Phase 4: Frontend Integration

#### 4.1 Update Chat Component

**File:** `components/chat.tsx`
- Add support for OpenAI models
- Enable reasoning visualization
- Maintain existing UI layout

```tsx
// Snippet of changes to components/chat.tsx
import { useState } from "react";
import { useChat } from "@ai-sdk/react";
import { models } from "@/lib/models";
// Other imports...

export function Chat() {
  const [input, setInput] = useState<string>("");
  const [selectedModelId] = useState<string>("gpt-4o"); // Default to GPT-4o
  const [isReasoningEnabled, setIsReasoningEnabled] = useState<boolean>(true);

  const selectedModel = models.find((model) => model.id === selectedModelId);

  const { messages, append, status, stop } = useChat({
    id: "primary",
    body: {
      selectedModelId,
      isReasoningEnabled,
    },
    onError: () => {
      toast.error("An error occurred, please try again!");
    },
  });

  // Rest of the component remains similar
  // ...
}
```

#### 4.2 Update Messages Component

**File:** `components/messages.tsx`
- Enhance for reasoning steps display
- Add citation rendering
- Maintain current UI aesthetic

```tsx
// Snippet of changes to components/messages.tsx
// Add support for citations and sources
interface SourceCitation {
  url: string;
  title?: string;
  provider: string;
}

interface MessageWithSources extends UIMessage {
  sources?: SourceCitation[];
}

// Add citation component
function Citation({ source }: { source: SourceCitation }) {
  return (
    <div className="text-xs text-zinc-500 mt-1 flex items-center">
      <a 
        href={source.url} 
        target="_blank" 
        rel="noopener noreferrer"
        className="underline hover:text-zinc-800 dark:hover:text-zinc-300 flex items-center"
      >
        {source.title || source.url}
        <span className="ml-1 text-[10px] bg-zinc-200 dark:bg-zinc-800 px-1 rounded">
          {source.provider}
        </span>
      </a>
    </div>
  );
}

// Update to support sources in the message content
export function MessageContent({ message }: { message: MessageWithSources }) {
  // Existing code...
  
  // Add sources if available
  return (
    <div>
      {/* Existing markdown rendering */}
      <Markdown components={markdownComponents}>
        {message.content}
      </Markdown>
      
      {/* Render sources if available */}
      {message.sources && message.sources.length > 0 && (
        <div className="mt-2">
          <div className="text-xs text-zinc-500 mb-1">Sources:</div>
          {message.sources.map((source, index) => (
            <Citation key={index} source={source} />
          ))}
        </div>
      )}
    </div>
  );
}
```

### Phase 5: Memory and Collaboration System

#### 5.1 Implement Memory System

**Files to create:**
- `lib/memory/index.ts`: Memory system implementation
- `lib/memory/types.ts`: Memory type definitions
- `lib/memory/storage.ts`: Storage implementation

```typescript
// lib/memory/types.ts example
export type MemoryType = "SESSION" | "USER_PREFERENCE" | "CONVERSATION" | "DOMAIN_KNOWLEDGE" | "TOOL_USAGE";

export interface MemoryEntry {
  id: string;
  type: MemoryType;
  content: string;
  metadata?: Record<string, any>;
  embedding?: number[];
  createdAt: Date;
  expiresAt?: Date;
}

// lib/memory/index.ts example
import { MemoryEntry, MemoryType } from "./types";
import { createMemoryStorage } from "./storage";

class MemorySystem {
  private storage = createMemoryStorage();

  async store(type: MemoryType, content: string, metadata?: Record<string, any>, ttlMs?: number): Promise<string> {
    const id = crypto.randomUUID();
    const entry: MemoryEntry = {
      id,
      type,
      content,
      metadata,
      createdAt: new Date(),
      expiresAt: ttlMs ? new Date(Date.now() + ttlMs) : undefined,
    };
    
    await this.storage.set(id, entry);
    return id;
  }

  async retrieve(id: string): Promise<MemoryEntry | undefined> {
    return this.storage.get(id);
  }

  async searchSimilar(content: string, type?: MemoryType, limit = 5): Promise<MemoryEntry[]> {
    // Implementation with embeddings would go here
    // For now, return recent entries of the specified type
    const entries = await this.storage.getAll();
    return entries
      .filter(entry => !type || entry.type === type)
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
      .slice(0, limit);
  }
}

export const memorySystem = new MemorySystem();
```

#### 5.2 Implement Collaboration Protocols

**Files to create:**
- `lib/agents/collaboration/sequential.ts`: Sequential execution
- `lib/agents/collaboration/parallel.ts`: Parallel execution
- `lib/agents/collaboration/debate.ts`: Debate protocol
- `lib/agents/collaboration/consensus.ts`: Consensus building

```typescript
// lib/agents/collaboration/sequential.ts example
import { Agent } from "../types";
import { Message } from "ai";
import { myProvider } from "@/lib/models";
import { generateText } from "ai";

export async function sequentialExecution(
  agents: Agent[],
  initialMessages: Message[],
  context?: Record<string, any>
): Promise<{ result: string; intermediateSteps: Array<{ agent: Agent; output: string }> }> {
  let currentMessages = [...initialMessages];
  const intermediateSteps: Array<{ agent: Agent; output: string }> = [];

  for (const agent of agents) {
    // Generate response from this agent
    const response = await generateText({
      model: myProvider.languageModel(agent.defaultModel),
      system: agent.systemPrompt,
      messages: currentMessages,
    });

    // Store the intermediate step
    intermediateSteps.push({
      agent,
      output: response.content,
    });

    // Add this response to the messages for the next agent
    currentMessages.push({
      role: "assistant",
      content: response.content,
    });
  }

  // The final result is the last agent's output
  return {
    result: intermediateSteps[intermediateSteps.length - 1].output,
    intermediateSteps,
  };
}
```

### Phase 6: Financial UI Components Integration

#### 6.1 Copy and Update UI Components

**Files to copy from gen-ui-finance/ to components/finance/:**
- `financials-table.tsx`
- `stock-chart.tsx`
- `stock-screener-table.tsx`
- `table.tsx`
- `ticker-suggestions.tsx`

**File to create:**
- `lib/tools/ui/index.ts`: UI tool integration

```typescript
// lib/tools/ui/index.ts example
import { z } from "zod";
import { toolRegistry } from "../registry";

// Register createDocument tool
toolRegistry.register({
  name: "createDocument",
  description: "Create a document for a writing activity.",
  parameters: z.object({
    title: z.string().describe("The title of the document"),
    kind: z.enum(["text", "code"]).describe("The kind of document to create"),
  }),
  execute: async (params) => {
    // Implementation would go here
    const id = crypto.randomUUID();
    return {
      id,
      title: params.title,
      kind: params.kind,
      content: "",
    };
  },
});

// Register other UI tools...
```

#### 6.2 Integrate Financial Components

**File to create:**
- `components/finance/index.tsx`: Component selection and rendering

```tsx
// components/finance/index.tsx example
import dynamic from "next/dynamic";
import { useState, useEffect } from "react";

// Dynamically import components to reduce initial bundle size
const FinancialsTable = dynamic(() => import("./financials-table"));
const StockChart = dynamic(() => import("./stock-chart"));
const StockScreenerTable = dynamic(() => import("./stock-screener-table"));
const TickerSuggestions = dynamic(() => import("./ticker-suggestions"));

export type FinanceComponentType = 
  | "financials-table"
  | "stock-chart"
  | "stock-screener-table"
  | "ticker-suggestions";

interface FinanceComponentProps {
  type: FinanceComponentType;
  data: any;
}

export function FinanceComponent({ type, data }: FinanceComponentProps) {
  switch (type) {
    case "financials-table":
      return <FinancialsTable data={data} />;
    case "stock-chart":
      return <StockChart data={data} />;
    case "stock-screener-table":
      return <StockScreenerTable data={data} />;
    case "ticker-suggestions":
      return <TickerSuggestions data={data} />;
    default:
      return <div>Unknown component type</div>;
  }
}
```

### Phase 12: Advanced Agentic Patterns from OpenAI

After reviewing OpenAI's Python agent patterns, we can enhance our implementation with several powerful agentic patterns that address specific use cases and improve overall system robustness.

#### 12.1 Deterministic Agentic Workflows

**Implement explicit validation steps between agent executions:**

```typescript
// lib/agents/workflows/deterministicFlow.ts
import { z } from 'zod';
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner } from '@/lib/agents/AgentRunner';
import { AgentError } from '@/lib/errors/AgentError';

// Define structured output for validation
const OutlineValidationSchema = z.object({
  good_quality: z.boolean(),
  is_financial_related: z.boolean(),
  reasoning: z.string()
});

type OutlineValidation = z.infer<typeof OutlineValidationSchema>;

// Create specialized agents for each step
export const outlineGeneratorAgent = new Agent({
  name: 'outline_generator',
  instructions: 'Generate a very short outline for financial analysis based on the user query.',
  model: 'gpt-4o-mini'
});

export const outlineValidatorAgent = new Agent({
  name: 'outline_validator',
  instructions: 'Evaluate the financial analysis outline for quality and relevance.',
  model: 'gpt-4o-mini'
});

export const analysisWriterAgent = new Agent({
  name: 'analysis_writer',
  instructions: 'Write a comprehensive financial analysis based on the provided outline.',
  model: 'gpt-4o'
});

// Deterministic workflow with validation gates
export async function deterministicFinancialAnalysis(query: string): Promise<string> {
  // Step 1: Generate outline
  const outline = await AgentRunner.run(outlineGeneratorAgent, [query]);
  console.log("Outline generated");
  
  // Step 2: Validate outline
  const validationResult = await AgentRunner.run(outlineValidatorAgent, [outline.agentOutput]);
  const validation = validationResult.agentOutput as OutlineValidation;
  
  // Step 3: Gate - only proceed if outline passes validation
  if (!validation.good_quality) {
    throw new AgentError(`Outline quality insufficient: ${validation.reasoning}`);
  }
  
  if (!validation.is_financial_related) {
    throw new AgentError(`Outline not related to financial analysis: ${validation.reasoning}`);
  }
  
  console.log("Outline validated, proceeding to full analysis");
  
  // Step 4: Write full analysis
  const analysis = await AgentRunner.run(analysisWriterAgent, [outline.agentOutput]);
  
  return analysis.agentOutput as string;
}
```

#### 12.2 Agents as Tools Pattern

**Encapsulate agents as reusable tools:**

```typescript
// lib/agents/AgentExtensions.ts
import { Agent } from '@/lib/agents/Agent';
import { tool } from 'ai';
import { z } from 'zod';

// Extend the Agent class with a method to convert it to a tool
export function agentAsTool(agent: Agent, toolName: string, toolDescription: string) {
  return tool({
    name: toolName,
    description: toolDescription,
    parameters: z.object({
      input: z.string().describe('The input to pass to the agent')
    }),
    execute: async ({ input }) => {
      const result = await agent.execute(input);
      return result.agentOutput;
    }
  });
}

// Usage example
// lib/tools/agentTools.ts
import { financialAnalystAgent, researchAgent } from '@/lib/agents/registry';
import { agentAsTool } from '@/lib/agents/AgentExtensions';

export const financialAnalysisTool = agentAsTool(
  financialAnalystAgent,
  'analyze_financials',
  'Analyze financial data and provide insights.'
);

export const researchTool = agentAsTool(
  researchAgent,
  'research_topic',
  'Research a topic and provide relevant information.'
);

// In the lead orchestrator agent definition
export const leadOrchestratorAgent = new Agent({
  name: 'lead_orchestrator',
  instructions: 'You are a lead orchestrator agent. Coordinate tasks between specialized agents.',
  tools: [financialAnalysisTool, researchTool],
  model: 'gpt-4o'
});
```

#### 12.3 LLM-as-a-Judge Pattern

**Implement iterative feedback loops for quality improvement:**

```typescript
// lib/agents/workflows/llmJudge.ts
import { z } from 'zod';
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner } from '@/lib/agents/AgentRunner';

// Define the evaluation schema
const EvaluationSchema = z.object({
  score: z.enum(['pass', 'needs_improvement', 'fail']),
  feedback: z.string()
});

type Evaluation = z.infer<typeof EvaluationSchema>;

// Create the judge agent
export const evaluatorAgent = new Agent({
  name: 'evaluator',
  instructions: 'You evaluate content and provide detailed feedback for improvement. Be thorough and critical.',
  model: 'gpt-4o'
});

// Iterative improvement workflow
export async function iterativeImprovement(
  contentAgent: Agent,
  prompt: string,
  maxIterations: number = 3
): Promise<{ content: string; iterations: number; finalEvaluation: Evaluation }> {
  let iterations = 0;
  let currentPrompt = prompt;
  let latestContent = '';
  let evaluation: Evaluation = { score: 'needs_improvement', feedback: 'Initial evaluation' };
  
  while (evaluation.score !== 'pass' && iterations < maxIterations) {
    // Generate content
    const contentResult = await AgentRunner.run(contentAgent, [currentPrompt]);
    latestContent = contentResult.agentOutput as string;
    
    // Evaluate content
    const evaluationResult = await AgentRunner.run(
      evaluatorAgent, 
      [`Please evaluate this content and provide feedback:\n\n${latestContent}`]
    );
    evaluation = evaluationResult.agentOutput as Evaluation;
    
    // If not passed, update prompt with feedback
    if (evaluation.score !== 'pass') {
      currentPrompt = `${prompt}\n\nPrevious attempt: ${latestContent}\n\nFeedback: ${evaluation.feedback}\n\nPlease improve based on this feedback.`;
      iterations++;
      console.log(`Iteration ${iterations}: Score ${evaluation.score}`);
    }
  }
  
  return {
    content: latestContent,
    iterations,
    finalEvaluation: evaluation
  };
}
```

#### 12.4 Parallelization for Latency and Quality

**Implement parallel agent executions:**

```typescript
// lib/agents/workflows/parallelExecution.ts
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner } from '@/lib/agents/AgentRunner';

// Create a selection agent
export const selectionAgent = new Agent({
  name: 'result_selector',
  instructions: 'You analyze multiple results and select the best one, explaining your choice.',
  model: 'gpt-4o-mini'
});

// Parallel execution with best result selection
export async function parallelAgentExecution(
  agent: Agent, 
  input: string, 
  numParallel: number = 3
): Promise<string> {
  // Run agent multiple times in parallel
  const promiseArray = Array(numParallel).fill(0).map(() => 
    AgentRunner.run(agent, [input])
  );
  
  const results = await Promise.all(promiseArray);
  const outputs = results.map((result, index) => 
    `Option ${index + 1}:\n${result.agentOutput}`
  );
  
  // If only one execution, return that
  if (numParallel === 1) {
    return results[0].agentOutput as string;
  }
  
  // Select the best result
  const selectionPrompt = `
    I have ${numParallel} different results for the prompt: "${input}"
    
    ${outputs.join('\n\n')}
    
    Please analyze these options and select the best one. Explain your choice.
  `;
  
  const selection = await AgentRunner.run(selectionAgent, [selectionPrompt]);
  
  return selection.agentOutput as string;
}
```

#### 12.5 Enhanced Guardrails with Tripwires

**Implement specialized guardrail agents with immediate interruption capabilities:**

```typescript
// lib/guardrails/tripwireGuardrails.ts
import { z } from 'zod';
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner } from '@/lib/agents/AgentRunner';
import { GuardrailError } from '@/lib/errors/AgentError';

// Define the guardrail result schema
const GuardrailResultSchema = z.object({
  passed: z.boolean(),
  reason: z.string(),
  confidence: z.number().min(0).max(1)
});

type GuardrailResult = z.infer<typeof GuardrailResultSchema>;

// Create a guardrail agent for financial homework detection
export const financialHomeworkGuardrailAgent = new Agent({
  name: 'financial_homework_guardrail',
  instructions: 'Detect if the user is asking for help with financial homework or academic assignments.',
  model: 'gpt-4o-mini'
});

// Guardrail function
export async function checkFinancialHomework(input: string): Promise<GuardrailResult> {
  const result = await AgentRunner.run(financialHomeworkGuardrailAgent, [input]);
  return result.agentOutput as GuardrailResult;
}

// Middleware to apply guardrails
export async function applyGuardrails(
  input: string, 
  guardrails: Array<(input: string) => Promise<GuardrailResult>>
) {
  for (const guardrail of guardrails) {
    const result = await guardrail(input);
    if (!result.passed) {
      throw new GuardrailError(`Guardrail triggered: ${result.reason}`, result);
    }
  }
  
  // If all guardrails pass, return the input
  return input;
}

// Usage in API route
// app/api/chat/route.ts
try {
  // Apply input guardrails first
  const validatedInput = await applyGuardrails(userInput, [
    checkFinancialHomework,
    // Add other guardrails here
  ]);
  
  // Continue with agent execution
  const result = await AgentRunner.run(leadOrchestratorAgent, [validatedInput]);
  
  // Return the result
  return new Response(JSON.stringify({ content: result.agentOutput }));
} catch (error) {
  if (error instanceof GuardrailError) {
    // Handle guardrail violations with appropriate response
    return new Response(JSON.stringify({ 
      error: "Policy violation", 
      message: "I'm sorry, I can't help with financial homework assignments." 
    }));
  }
  // Handle other errors
}
```

#### 12.6 Dynamic Handoffs and Routing

**Implement a frontline orchestrator with dynamic routing:**

```typescript
// lib/agents/workflows/dynamicRouting.ts
import { z } from 'zod';
import { Agent } from '@/lib/agents/Agent';
import { AgentRunner } from '@/lib/agents/AgentRunner';
import { AgentError } from '@/lib/errors/AgentError';
import { 
  financialAnalystAgent, 
  researchAgent,
  marketDataAgent,
  stockScreenerAgent 
} from '@/lib/agents/registry';

// Define the routing decision schema
const RoutingDecisionSchema = z.object({
  agent_type: z.enum([
    'financial_analyst', 
    'researcher', 
    'market_data',
    'stock_screener'
  ]),
  confidence: z.number().min(0).max(1),
  reasoning: z.string()
});

type RoutingDecision = z.infer<typeof RoutingDecisionSchema>;

// Create a routing agent
export const routingAgent = new Agent({
  name: 'routing_agent',
  instructions: `
    You determine the most appropriate specialized agent to handle the user query.
    Options:
    - financial_analyst: For financial analysis, investment advice, and company evaluations
    - researcher: For general information gathering or factual questions
    - market_data: For specific stock price or market index data queries
    - stock_screener: For finding stocks matching specific criteria
  `,
  model: 'gpt-4o-mini'
});

// Dynamic routing workflow
export async function dynamicRouting(query: string): Promise<string> {
  // Determine the appropriate agent
  const routingResult = await AgentRunner.run(routingAgent, [query]);
  const decision = routingResult.agentOutput as RoutingDecision;
  
  console.log(`Routing to: ${decision.agent_type} (confidence: ${decision.confidence})`);
  
  // Confidence threshold - if below this, use the most comprehensive agent
  if (decision.confidence < 0.7) {
    console.log(`Low confidence routing (${decision.confidence}), using financial analyst`);
    return await AgentRunner.run(financialAnalystAgent, [query]).then(res => res.agentOutput as string);
  }
  
  // Route to the appropriate agent
  switch (decision.agent_type) {
    case 'financial_analyst':
      return await AgentRunner.run(financialAnalystAgent, [query]).then(res => res.agentOutput as string);
    case 'researcher':
      return await AgentRunner.run(researchAgent, [query]).then(res => res.agentOutput as string);
    case 'market_data':
      return await AgentRunner.run(marketDataAgent, [query]).then(res => res.agentOutput as string);
    case 'stock_screener':
      return await AgentRunner.run(stockScreenerAgent, [query]).then(res => res.agentOutput as string);
    default:
      throw new AgentError(`Unsupported agent type: ${decision.agent_type}`);
  }
}
```

#### 12.7 Combined Workflow Strategy

Combining these patterns strategically creates a comprehensive agentic workflow:

```typescript
// lib/workflows/comprehensiveWorkflow.ts
import { AgentError, GuardrailError } from '@/lib/errors/AgentError';
import { applyGuardrails, checkFinancialHomework } from '@/lib/guardrails/tripwireGuardrails';
import { dynamicRouting } from '@/lib/agents/workflows/dynamicRouting';
import { parallelAgentExecution } from '@/lib/agents/workflows/parallelExecution';
import { iterativeImprovement } from '@/lib/agents/workflows/llmJudge';
import { leadOrchestratorAgent } from '@/lib/agents/registry';

export async function comprehensiveFinancialWorkflow(query: string) {
  try {
    // Step 1: Apply input guardrails
    await applyGuardrails(query, [checkFinancialHomework]);
    
    // Step 2: Dynamic routing to appropriate agent
    const routingResult = await dynamicRouting(query);
    
    // Step 3: For important tasks, use parallel execution for best results
    let initialResult;
    if (query.includes('investment') || query.includes('portfolio')) {
      initialResult = await parallelAgentExecution(leadOrchestratorAgent, query, 3);
    } else {
      initialResult = routingResult;
    }
    
    // Step 4: Apply iterative improvement with LLM-as-a-judge
    const finalResult = await iterativeImprovement(
      leadOrchestratorAgent,
      `Original query: ${query}\nInitial response: ${initialResult}\nImprove this response.`,
      2
    );
    
    return finalResult.content;
    
  } catch (error) {
    if (error instanceof GuardrailError) {
      return `I cannot assist with that request because ${error.message}`;
    }
    if (error instanceof AgentError) {
      return `I encountered an issue while processing your request: ${error.message}`;
    }
    throw error; // Rethrow unexpected errors
  }
}
```

## Additional Verification Checklist Items

Add these items to the Implementation Verification Checklist:

- [ ] **Advanced Agentic Patterns**
  - [ ] Deterministic workflows with validation gates implemented
  - [ ] Agents-as-tools pattern for modularity and reuse
  - [ ] LLM-as-a-judge iterative improvement loops
  - [ ] Parallel execution for latency optimization and quality
  - [ ] Tripwire guardrails for immediate interruption
  - [ ] Dynamic routing between specialized agents
  - [ ] Combined workflow strategies for comprehensive tasks

## Implementation Timeline

### Week 1: Foundation
- Update model system
- Create tool registry
- Basic OpenAI Responses API integration
- Set up testing infrastructure

### Week 2: Search and Financial
- Implement search provider integrations
- Add financial tools
- Create memory system basics
- Update frontend for reasoning

### Week 3: Advanced Features
- Implement agent registry and orchestrator
- Add collaboration protocols
- Integrate financial UI components
- Implement dynamic UI generation

### Week 4: Refinement
- Comprehensive testing
- Performance optimization
- Documentation updates
- Final integration and deployment

## Implementation Verification Checklist

- [ ] **Model Integration**
  - [ ] OpenAI models configured in lib/models.ts
  - [ ] Feature flagging implemented for model capabilities
  - [ ] Reasoning controls with object-based format
  - [ ] Provider-specific options for OpenAI

- [ ] **Tool Registry**
  - [ ] Central tool registry created
  - [ ] Tool type definitions and validation
  - [ ] Search tools integrated
  - [ ] Financial tools integrated
  - [ ] UI tools integrated

- [ ] **API Routing**
  - [ ] app/api/chat/route.ts updated for Responses API
  - [ ] Lead orchestrator integration
  - [ ] Multi-step processing pipeline
  - [ ] Reasoning configuration

- [ ] **Agent System**
  - [ ] Agent registry with domain and role specializations
  - [ ] Lead orchestrator implementation
  - [ ] Domain-specific agents created
  - [ ] Collaboration protocols implemented

- [ ] **Frontend Components**
  - [ ] components/chat.tsx updated for OpenAI models
  - [ ] components/messages.tsx enhanced for reasoning
  - [ ] Citation and source rendering
  - [ ] Dynamic UI component generation

- [ ] **Memory System**
  - [ ] Memory type definitions
  - [ ] Storage implementation
  - [ ] Retrieval strategies
  - [ ] Memory integration with chat session

- [ ] **Financial UI Components**
  - [ ] UI components copied and updated
  - [ ] Component selection mechanism
  - [ ] Financial data visualization
  - [ ] Tool connection with UI components

- [ ] **Testing and Performance**
  - [ ] Unit tests for key components
  - [ ] Integration tests for full flow
  - [ ] Performance metrics collected
  - [ ] Optimizations implemented 