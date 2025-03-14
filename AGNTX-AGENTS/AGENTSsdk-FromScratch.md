<?xml version="1.0" encoding="UTF-8"?>
<guide title="Comprehensive Guide: Recreating OpenAI Agents SDK using Vercel AI SDK & OpenAI Responses API">
  <section id="introduction">
    <title>Introduction</title>
    <description>
      This guide demonstrates how to implement the key features of the OpenAI Agents SDK—
      Agents, Tools, Context, Output Types, Handoffs, Streaming, Tracing, and Guardrails—
      using only the Vercel AI SDK (ai) and the OpenAI Responses API (@ai-sdk/openai) in TypeScript.
      It emphasizes strong type safety with Zod, robust error handling, and a modular design.
    </description>
  </section>
  
  <section id="agents">
    <title>1. Defining Agents</title>
    <description>
      Agents encapsulate instructions, tools, and model settings for task execution.
      The Agent class stores a name, instructions (which serve as the system prompt), tools, and the model.
    </description>
    <code language="typescript"><![CDATA[
import { openai } from '@ai-sdk/openai';
import { ToolFunction } from 'ai';

export interface AgentConfig<Ctx = any, Output = string> {
  name: string;
  instructions: string;
  tools?: ToolFunction<any, any>[];
  model?: string;
}

export class Agent<Ctx = any, Output = string> {
  name: string;
  instructions: string;
  tools: ToolFunction<any, any>[];
  model: string;

  constructor({ name, instructions, tools = [], model = 'gpt-4o' }: AgentConfig<Ctx, Output>) {
    this.name = name;
    this.instructions = instructions;
    this.tools = tools;
    this.model = model;
  }

  getSystemPrompt(): string {
    return this.instructions;
  }
}
    ]]></code>
  </section>
  
  <section id="tools">
    <title>2. Tool Definitions</title>
    <description>
      Tools are external functions the agent can call.
      We use the Vercel AI SDK’s native tool() helper with Zod schemas for input validation.
    </description>
    <code language="typescript"><![CDATA[
import { z } from 'zod';
import { tool } from 'ai';

export const getWeatherTool = tool({
  description: 'Get weather for a specified location.',
  parameters: z.object({
    location: z.string(),
    unit: z.enum(['C', 'F']).default('C')
  }),
  execute: async ({ location, unit }) => {
    // Simulate or call an external weather API
    return `Weather in ${location}: 22°${unit}`;
  }
});
    ]]></code>
    <description>
      A helper function to convert custom tools into the format required by generateText.
    </description>
    <code language="typescript"><![CDATA[
import { tool, ToolFunction } from 'ai';

export interface MyTool<Ctx> {
  name: string;
  description: string;
  parameters: z.ZodType<any>;
  execute: (args: any, context: Ctx) => Promise<any>;
}

export function makeToolFunctions<Ctx>(tools: MyTool<Ctx>[], context: Ctx): Record<string, ToolFunction<any, any>> {
  const toolMap: Record<string, ToolFunction<any, any>> = {};
  for (const t of tools) {
    toolMap[t.name] = tool({
      description: t.description,
      parameters: t.parameters,
      execute: async (args) => t.execute(args, context)
    });
  }
  return toolMap;
}
    ]]></code>
  </section>
  
  <section id="context">
    <title>3. Context Handling</title>
    <description>
      Context is a mutable object holding session data or shared resources across tool calls.
      Use a strongly typed interface and pass it to your tools.
    </description>
    <code language="typescript"><![CDATA[
interface SessionContext {
  userId: string;
  data: Record<string, any>;
}

const context: SessionContext = {
  userId: 'user123',
  data: {}
};

const storeDataTool = tool({
  description: 'Stores data into the context.',
  parameters: z.object({ key: z.string(), value: z.any() }),
  execute: async ({ key, value }) => {
    context.data[key] = value;
    return `Stored ${key}`;
  }
});
    ]]></code>
  </section>
  
  <section id="output">
    <title>4. Structured Output Types</title>
    <description>
      Use structured outputs by instructing the model to call a dedicated final_output tool.
      We define a Zod schema and use it for validation.
    </description>
    <code language="typescript"><![CDATA[
import { z } from 'zod';
import { tool } from 'ai';

const finalOutputSchema = z.object({
  answer: z.string(),
  confidence: z.number().min(0).max(1)
});

export const finalOutputTool = tool({
  description: 'Final structured output.',
  parameters: finalOutputSchema,
  execute: async (output) => output
});
    ]]></code>
  </section>
  
  <section id="handoffs">
    <title>5. Handoffs</title>
    <description>
      Handoffs delegate tasks to another agent.
      A handoff tool can call a sub-agent and return its output.
    </description>
    <code language="typescript"><![CDATA[
const spanishAgent = new Agent({
  name: 'spanish_agent',
  instructions: 'Respond strictly in Spanish.'
});

const handoffToSpanishTool = tool({
  description: 'Delegates conversation to a Spanish-speaking agent.',
  parameters: z.object({ input: z.string() }),
  execute: async ({ input }) => {
    const result = await AgentRunner.run(spanishAgent, [input]);
    return result.agentOutput;
  }
});
    ]]></code>
  </section>
  
  <section id="streaming">
    <title>6. Streaming</title>
    <description>
      Streaming enables real-time partial outputs.
      Use streamText to yield text deltas as the model processes the prompt.
    </description>
    <code language="typescript"><![CDATA[
import { streamText } from 'ai';

const streamAgentOutput = async (agent: Agent, prompt: string) => {
  const stream = await streamText({
    model: openai(agent.model),
    system: agent.getSystemPrompt(),
    prompt,
    tools: agent.tools,
    maxSteps: 10
  });
  
  for await (const delta of stream.textStream) {
    process.stdout.write(delta);
  }
};
    ]]></code>
  </section>
  
  <section id="tracing">
    <title>7. Tracing & Logging</title>
    <description>
      Capture execution traces using callbacks such as onStepFinish.
      This helps debug and analyze agent runs.
    </description>
    <code language="typescript"><![CDATA[
const logs: Array<{ step: number; text: string; toolCalls?: any }> = [];

const result = await generateText({
  model: openai(agent.model),
  system: agent.getSystemPrompt(),
  prompt: 'What is the capital of Spain?',
  tools: agent.tools,
  maxSteps: 10,
  onStepFinish: ({ step, text, toolCalls }) => {
    logs.push({ step, text, toolCalls });
  }
});

console.log('Trace:', logs);
    ]]></code>
  </section>
  
  <section id="guardrails">
    <title>8. Guardrails</title>
    <description>
      Guardrails validate input and output to enforce policies (e.g., no profanity).
      They run before and after agent execution.
    </description>
    <code language="typescript"><![CDATA[
type GuardrailResult = { passed: boolean; details?: string };

interface Guardrail {
  name: string;
  validate: (content: string[], ctx?: any) => Promise<GuardrailResult>;
}

const profanityGuardrail: Guardrail = {
  name: 'no_profanity',
  validate: async (messages) => {
    const hasProfanity = messages.some((msg) => /badword/.test(msg));
    return hasProfanity
      ? { passed: false, details: 'Profanity detected.' }
      : { passed: true };
  }
};

const applyGuardrails = async (guardrails: Guardrail[], messages: string[]) => {
  for (const guardrail of guardrails) {
    const result = await guardrail.validate(messages);
    if (!result.passed) throw new Error(`Guardrail Failed: ${result.details}`);
  }
};
    ]]></code>
  </section>
  
  <section id="runner">
    <title>9. Agent Runner</title>
    <description>
      AgentRunner integrates all features: it applies guardrails, wraps tools,
      runs generateText in multi-step mode, and optionally parses structured output.
    </description>
    <code language="typescript"><![CDATA[
import { generateText } from 'ai';

export class AgentRunner {
  static async run<Ctx, Out>(
    agent: Agent<Ctx, Out>,
    inputs: string[],
    context?: Ctx,
    guardrails?: { input?: Guardrail[]; output?: Guardrail[] }
  ): Promise<{ agentOutput: Out }> {
    // Apply input guardrails if provided
    if (guardrails?.input) await applyGuardrails(guardrails.input, inputs);

    // Convert agent tools with the helper
    const toolMap = makeToolFunctions(agent.tools, context);

    const result = await generateText({
      model: openai(agent.model),
      system: agent.getSystemPrompt(),
      prompt: inputs.join('\n'),
      tools: toolMap,
      maxSteps: 10,
      onStepFinish: ({ text, toolCalls }) => {
        // Optional tracing or logging
      }
    });

    let finalStructuredOutput: Out | undefined;

    // Look for a final_output tool call among steps
    for (const step of result.steps) {
      for (const call of step.toolCalls || []) {
        if (call.toolName === 'final_output') {
          finalStructuredOutput = finalOutputSchema.parse(call.result) as Out;
          break;
        }
      }
      if (finalStructuredOutput) break;
    }

    const finalOutput = finalStructuredOutput ?? (result.text as Out);

    // Apply output guardrails if provided
    if (guardrails?.output) await applyGuardrails(guardrails.output, [JSON.stringify(finalOutput)]);

    return { agentOutput: finalOutput };
  }
}
    ]]></code>
  </section>
  
  <section id="error-handling">
    <title>10. Error Handling</title>
    <description>
      Handle errors robustly using custom exceptions. This prevents unexpected failures.
    </description>
    <code language="typescript"><![CDATA[
export class AgentError extends Error {
  constructor(message: string, public cause?: unknown) {
    super(message);
    this.name = 'AgentError';
  }
}

// Usage within tool execution:
try {
  const result = await someTool.execute(args);
} catch (e) {
  throw new AgentError(`Error executing tool ${someTool.name}`, e);
}
    ]]></code>
  </section>
  
  <section id="conclusion">
    <title>Conclusion & Best Practices</title>
    <description>
      This guide demonstrates how to recreate the OpenAI Agents SDK features using only the Vercel AI SDK and OpenAI Responses API.
      It covers:
      - Agents: Configured with instructions, tools, and model settings.
      - Tools: Defined with tool() and Zod validation.
      - Context: Managed as a strongly typed mutable object.
      - Structured Output: Enforced via a final_output tool.
      - Handoffs: Delegated via specialized tools.
      - Streaming: Implemented with streamText.
      - Tracing: Captured via callbacks.
      - Guardrails: Input/output validation.
      - Error Handling: Using custom exceptions.
      This modular, robust design aligns with production best practices and leverages the full capabilities of the Vercel AI SDK.
    </description>
  </section>
</guide>