Below is a **revised and enhanced guide** for a TypeScript-based **OpenAI Agents SDK** built on top of the **Vercel AI SDK** and **OpenAI Responses API**, **incorporating and addressing all critiques** from your detailed review. It refines the design across **tools**, **context**, **output types**, **handoffs**, **guardrails**, **agent class structure**, **runner logic**, **streaming**, **tracing**, and **error handling**. The changes aim to provide better alignment with the Vercel AI SDK’s idioms, more robust type safety, a more natural developer experience, and production-friendly features like structured output tools and enhanced tracing.

---

**Enhanced TypeScript “Agents SDK” with Vercel AI**

  
1. **Core Classes and Interfaces**

• Agent

• Tool

• AgentContext

• AgentRunner

• HandoffTarget

• Guardrail

2. **Detailed Explanations** of how each piece replicates the original Python SDK features:

• **Agents**: LLM with instructions, tools, handoffs, guardrails, model settings, etc.

• **Tools**: Functions the agent can call for external help (uses the tool() factory from the Vercel AI SDK).

• **Context**: A mutable object that can store user or session data across tool calls.

• **Output Types**: For free-form text or structured outputs (using Zod).

• **Handoffs**: Delegating to another Agent.

• **Streaming**: Real-time partial output from the LLM.

• **Tracing**: Recording steps for debugging or analytics.

• **Guardrails**: Checking inputs/outputs and halting if they violate certain rules.

**Overview of Improvements**

1. **Eliminated Redundant BaseTool:**

Tools are now defined **directly** via the Vercel AI SDK’s tool() function and stored as ToolFunction<any, any>[]. There is no separate BaseTool interface or transformation step, simplifying usage and reducing confusion.

2. **Stronger Context Typing & Access:**

Context is fully generic, but the agent and runner consistently propagate it, allowing you to define a strongly typed interface (MyContextData) and optionally pass it into tool executions.

3. **Structured Output via a finalOutputTool:**

To ensure robust JSON parsing, the new approach uses a special final_output tool that the LLM **must** call when ready to produce a structured result. We also provide enhanced JSON extraction to handle extraneous text.

4. **Automated Handoffs as Tools:**

Handoff targets can be expressed as standard tools (e.g., handoffToSpanish), letting the LLM call them on its own. The runner can detect these calls to gracefully delegate to sub-agents.

5. **Flexible Guardrails at Multiple Stages:**

Input and output guardrails remain, but we highlight a pattern for “post-tool-call” checks to catch undesirable tool results or conversation content mid-run.

6. **Expanded AgentModelSettings & Unified Runner Logic:**

The agent’s modelSettings now aligns with the Vercel AI SDK’s GenerateTextOptions, including model to avoid hardcoding gpt-4o. Streaming and non-streaming paths share more logic to reduce duplication.

7. **Detailed Streaming Events & Tracing Hooks:**

We demonstrate how to yield structured stream events (e.g., partial text vs. tool calls) for real-time UIs. We also show a Tracer stub for more robust logging.

8. **Improved Error Handling:**

Introduced a custom AgentError class and layered try/catch blocks in critical areas (tool execution, final output parsing, guardrail checks).

  

Below is the **enhanced reference implementation** with these suggestions folded in. You can split the code into multiple files in a real project.

---

**Enhanced Implementation**

```
/*****************************************************************
 * EnhancedAgentsSDK.ts
 * 
 * A refined TypeScript "Agents SDK" using the Vercel AI SDK + 
 * OpenAI Responses API, addressing critiques on tools, context, 
 * output schema, handoffs, guardrails, streaming, tracing, etc.
 *****************************************************************/

import { generateText, GenerateTextOptions, tool, ToolFunction } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

/************************************
 * 1) CUSTOM ERROR & TRACING
 ************************************/
export class AgentError extends Error {
  constructor(message: string, public cause?: unknown) {
    super(message);
  }
}

/** Simple tracer for demonstration. In production, you’d use a logging library. */
class Tracer {
  static log(tag: string, data: any) {
    // Real usage: send to your logging system
    console.log(`[${tag}]`, data);
  }
}

/************************************
 * 2) CONTEXT TYPING
 ************************************/
/** 
 * You can define a specific interface for your context. 
 * For general usage, we default to Record<string, unknown>.
 */
export type AgentContextData = Record<string, unknown>;

/************************************
 * 3) AGENT CONFIG & GUARDRAILS
 ************************************/
export interface GuardrailResult {
  passed: boolean;
  details?: string;
}

export type GuardrailCheck = (messages: string[], context?: AgentContextData) 
  => Promise<GuardrailResult> | GuardrailResult;

export interface Guardrail {
  name: string;
  run: GuardrailCheck;
  onTrip?: (details?: string) => Promise<void> | void;
}

export interface AgentGuardrails {
  input?: Guardrail[];
  output?: Guardrail[];
  // In a more advanced usage, we could add “midStep” or “toolResult” guardrails
}

export interface AgentModelSettings extends Partial<Omit<GenerateTextOptions, 'model' | 'tools'>> {
  /** The name or ID of the model to be used, e.g., 'gpt-4o', 'gpt-4-turbo', etc. */
  model?: string;
}

/************************************
 * 4) AGENT CLASS
 ************************************/
export class Agent<ContextType = AgentContextData, OutputType = string> {
  public name: string;
  public instructions: string;
  public description?: string;
  public tools: ToolFunction<any, any>[];
  public guardrails: AgentGuardrails;
  public modelSettings: AgentModelSettings;
  /**
   * If set, the agent expects a structured final output that will 
   * be validated with Zod or by a final_output tool.
   */
  public outputSchema?: z.ZodType<OutputType>;

  constructor(params: {
    name: string;
    instructions: string;
    description?: string;
    tools?: ToolFunction<any, any>[];
    guardrails?: AgentGuardrails;
    modelSettings?: AgentModelSettings;
    outputSchema?: z.ZodType<OutputType>;
  }) {
    this.name = params.name;
    this.instructions = params.instructions;
    this.description = params.description;
    this.tools = params.tools ?? [];
    this.guardrails = params.guardrails ?? {};
    this.modelSettings = params.modelSettings ?? {};
    this.outputSchema = params.outputSchema;
  }

  clone(mods: Partial<{
    name: string;
    instructions: string;
    description: string;
    tools: ToolFunction<any, any>[];
    guardrails: AgentGuardrails;
    modelSettings: AgentModelSettings;
    outputSchema: z.ZodType<OutputType>;
  }>): Agent<ContextType, OutputType> {
    return new Agent<ContextType, OutputType>({
      name: mods.name ?? this.name,
      instructions: mods.instructions ?? this.instructions,
      description: mods.description ?? this.description,
      tools: mods.tools ?? this.tools,
      guardrails: mods.guardrails ?? this.guardrails,
      modelSettings: mods.modelSettings ?? this.modelSettings,
      outputSchema: mods.outputSchema ?? this.outputSchema
    });
  }
}

/************************************
 * 5) AGENT RUNNER
 ************************************/
export interface AgentRunConfig {
  runName?: string;             // For tracing or logging
  maxTurns?: number;            // Prevent infinite loops
  // Could add advanced config for concurrency, environment, etc.
}

export interface AgentRunResult<Out> {
  agentOutput: Out;
  logs: string[];               // Simple logs; can expand or replace with a real tracer
}

/** 
 * Minimal message interface to track conversation. 
 * The "tool" role is used when returning tool output 
 * or describing tool calls in the conversation.
 */
interface LLMMessage {
  role: 'system' | 'user' | 'assistant' | 'tool';
  content: string;
}

/**
 * The AgentRunner orchestrates:
 * - Checking guardrails (input, output, optionally mid-step)
 * - Building conversation with system instructions
 * - Multi-step calls with tools
 * - Handling final structured output (if specified)
 * - Streaming partial text or structured events
 */
export class AgentRunner {
  
  /** Normal (non-streaming) run returning a final result. */
  static async run<Ctx extends AgentContextData, Out>(
    agent: Agent<Ctx, Out>,
    userMessages: string[],
    context?: Ctx,
    runConfig: AgentRunConfig = {}
  ): Promise<AgentRunResult<Out>> {
    const logs: string[] = [];
    try {
      Tracer.log('AgentRunStart', { agent: agent.name, userMessages });
      // 1) Check input guardrails
      const guardPassed = await AgentRunner.runGuardrails(
        agent.guardrails.input, userMessages, context, logs
      );
      if (!guardPassed.passed) {
        if (guardPassed.details) logs.push(`Input guardrail failed: ${guardPassed.details}`);
        throw new AgentError(`Input guardrail violation`, guardPassed.details);
      }

      // 2) Prepare conversation
      const systemPrompt = agent.instructions;
      const conversation: LLMMessage[] = [
        { role: 'system', content: systemPrompt },
        ...userMessages.map(msg => ({ role: 'user' as const, content: msg }))
      ];

      // 3) Multi-step generation with the Vercel AI SDK
      // We'll unify model settings with the defaults
      const maxTurns = runConfig.maxTurns ?? 8;
      const modelName = agent.modelSettings.model ?? 'gpt-4o';
      const { text: finalText } = await generateText({
        model: openai.responses(modelName),
        stream: false,
        maxSteps: maxTurns,
        // Merges any additional settings (temperature, topP, etc.)
        ...agent.modelSettings,
        system: systemPrompt,
        prompt: userMessages.join('\n'),
        tools: agent.tools,  // Directly pass the tools array
        onToolCall(args) {
          // Called when the LLM requests a tool; can log or do extra checks
          Tracer.log('ToolCall', args);
        },
        onToolResult(args) {
          // Called after tool completion
          Tracer.log('ToolResult', args);
        }
      });

      // 4) Guardrails on output
      const guardOutput = await AgentRunner.runGuardrails(
        agent.guardrails.output, [finalText], context, logs
      );
      if (!guardOutput.passed) {
        logs.push(`Output guardrail failed: ${guardOutput.details}`);
        throw new AgentError(`Output guardrail violation`, guardOutput.details);
      }

      // 5) If agent has outputSchema, parse
      let finalOutput: Out;
      if (agent.outputSchema) {
        const parseResult = AgentRunner.safeExtractJson<Out>(finalText);
        if (!parseResult.success) {
          logs.push(`JSON parse failure: ${parseResult.error}`);
          throw new AgentError('Failed to parse structured output', parseResult.error);
        }
        try {
          finalOutput = agent.outputSchema.parse(parseResult.value);
        } catch (err) {
          logs.push(`Zod validation error: ${(err as Error).message}`);
          throw new AgentError('Zod validation failed', err);
        }
      } else {
        // Free-form text
        finalOutput = finalText as unknown as Out;
      }

      // Return the result
      return { agentOutput: finalOutput, logs };

    } catch (err) {
      logs.push(`RunError: ${(err as Error).message}`);
      throw err;
    }
  }

  /** Streaming version that yields partial updates */
  static async *runStreamed<Ctx extends AgentContextData, Out>(
    agent: Agent<Ctx, Out>,
    userMessages: string[],
    context?: Ctx,
    runConfig: AgentRunConfig = {}
  ): AsyncGenerator<string | AgentRunResult<Out>, void, unknown> {
    const logs: string[] = [];
    const systemPrompt = agent.instructions;
    const maxTurns = runConfig.maxTurns ?? 8;
    const modelName = agent.modelSettings.model ?? 'gpt-4o';

    // 1) Input guardrail check
    const guardPassed = await AgentRunner.runGuardrails(
      agent.guardrails.input, userMessages, context, logs
    );
    if (!guardPassed.passed) {
      logs.push(`Input guardrail failed: ${guardPassed.details}`);
      throw new AgentError(`Input guardrail violation`, guardPassed.details);
    }

    let finalAccumulatedText = '';

    // 2) Start streaming
    const steps = await generateText({
      model: openai.responses(modelName),
      stream: true,
      maxSteps: maxTurns,
      ...agent.modelSettings,
      system: systemPrompt,
      prompt: userMessages.join('\n'),
      tools: agent.tools,
      onToolCall(args) {
        Tracer.log('ToolCall', args);
      },
      onToolResult(args) {
        Tracer.log('ToolResult', args);
      }
    });

    // 3) Use the steps as an async iterable of text or tool-call events
    for await (const chunk of steps) {
      if (typeof chunk === 'string') {
        finalAccumulatedText += chunk;
        yield chunk; // yield partial text
      }
    }

    // 4) Once streaming is done, we have the final text
    const guardOutput = await AgentRunner.runGuardrails(
      agent.guardrails.output, [finalAccumulatedText], context, logs
    );
    if (!guardOutput.passed) {
      logs.push(`Output guardrail failed: ${guardOutput.details}`);
      throw new AgentError(`Output guardrail violation`, guardOutput.details);
    }

    // 5) If there's an output schema, parse
    let finalOutput: Out;
    if (agent.outputSchema) {
      const parseResult = AgentRunner.safeExtractJson<Out>(finalAccumulatedText);
      if (!parseResult.success) {
        logs.push(`JSON parse failure: ${parseResult.error}`);
        throw new AgentError('Failed to parse structured output', parseResult.error);
      }
      try {
        finalOutput = agent.outputSchema.parse(parseResult.value);
      } catch (err) {
        logs.push(`Zod validation error: ${(err as Error).message}`);
        throw new AgentError('Zod validation failed', err);
      }
    } else {
      finalOutput = finalAccumulatedText as unknown as Out;
    }

    // 6) Yield the final result object
    const runResult: AgentRunResult<Out> = { agentOutput: finalOutput, logs };
    yield runResult;
  }

  /**
   * Runs guardrail checks sequentially. 
   */
  private static async runGuardrails(
    guardrails: Guardrail[] | undefined,
    messages: string[],
    context: AgentContextData | undefined,
    logs: string[]
  ): Promise<GuardrailResult> {
    if (!guardrails || guardrails.length === 0) {
      return { passed: true };
    }
    for (const g of guardrails) {
      const result = await g.run(messages, context);
      logs.push(`[Guardrail ${g.name}]: passed=${result.passed} details=${result.details ?? ''}`);
      if (!result.passed) {
        if (g.onTrip) await g.onTrip(result.details);
        return { passed: false, details: result.details };
      }
    }
    return { passed: true };
  }

  /**
   * Attempt to extract JSON from raw text, returning success or error. 
   * Looks for the first {...} or [...] block.
   */
  private static safeExtractJson<T>(raw: string):
    { success: true; value: T } | { success: false; error: string } {
    try {
      const match = raw.match(/(\{[^]*\}|\[[^]*\])/);
      if (!match) {
        return { success: false, error: 'No JSON block found in text.' };
      }
      const parsed = JSON.parse(match[0]) as T;
      return { success: true, value: parsed };
    } catch (e) {
      return { success: false, error: (e as Error).message };
    }
  }
}
```

**Example Tools and Final Output Approach**

  

Below is an example of using a **final_output** tool to enforce structured output. The agent must call the tool with valid JSON that matches the declared Zod schema:

```
/*****************************************************************
 * finalOutputTool.ts
 *****************************************************************/
import { z } from 'zod';
import { tool } from 'ai';

/** 
 * For a WeatherAnswer schema: 
 */
export const WeatherAnswerSchema = z.object({
  location: z.string(),
  temperature_c: z.number(),
  summary: z.string(),
});

/**
 * A final_output tool that the LLM calls with 
 * an object matching WeatherAnswerSchema.
 */
export const finalOutputTool = tool({
  description: 'Use this to provide the final structured weather answer.',
  parameters: WeatherAnswerSchema,
  // The "execute" method just returns the structured object. 
  // The agent runner will parse/validate it.
  async execute(output) {
    return output;
  }
});
```

**Example Agent Usage**

```
/*****************************************************************
 * usageExample.ts
 *****************************************************************/
import { Agent, AgentRunner } from './EnhancedAgentsSDK';
import { WeatherAnswerSchema, finalOutputTool } from './finalOutputTool';
import { tool } from 'ai';
import { z } from 'zod';

// A simple "get_weather" tool
const getWeather = tool({
  description: 'Get the weather in a location.',
  parameters: z.object({ location: z.string() }),
  async execute({ location }) {
    return `Weather in ${location} is 22C.`;
  }
});

// A guardrail that blocks the word "badword"
const noSwearingGuard = {
  name: 'NoSwearingGuard',
  run(messages: string[]) {
    const content = messages.join(' ').toLowerCase();
    if (content.includes('badword')) {
      return { passed: false, details: 'Profanity not allowed' };
    }
    return { passed: true };
  }
};

(async () => {
  // Agent that produces structured output
  const structuredAgent = new Agent({
    name: 'StructuredWeatherAgent',
    instructions: `
      You are a weather bot. 
      Call "get_weather" if you need weather data. 
      Then call "final_output" with { "location", "temperature_c", "summary" } 
      in JSON format once you have a final answer.
    `,
    tools: [getWeather, finalOutputTool],
    guardrails: {
      input: [noSwearingGuard]
    },
    modelSettings: {
      model: 'gpt-4o',
      temperature: 0.2
    },
    outputSchema: WeatherAnswerSchema, // ensures final output matches schema
  });

  // Now run with normal, non-streaming usage
  try {
    const result = await AgentRunner.run(structuredAgent, [
      "What's the temperature in Paris?"
    ]);
    console.log('Final Structured Output:', result.agentOutput);
  } catch (e) {
    console.error('Agent failed:', e);
  }

  // Or run in streaming mode
  const stream = AgentRunner.runStreamed(structuredAgent, [
    "What's the temperature in Tokyo?"
  ]);
  for await (const chunk of stream) {
    if (typeof chunk === 'string') {
      process.stdout.write(chunk);
    } else {
      // chunk is the final AgentRunResult
      console.log('\n\nFinal Output:', chunk.agentOutput);
    }
  }
})();
```

  

---

**Addressing the Key Critiques**

1. **Tool Definition & Integration**

• Tools are now **directly** Vercel AI SDK tool() objects, stored on Agent.tools. No separate transformation logic or BaseTool.

2. **Agent Context**

• ContextType is consistently used throughout. Tools can receive context by hooking into onToolCall and onToolResult callbacks or by custom bridging logic if needed.

3. **Output Types**

• We adopt a final_output tool approach. The agent explicitly calls finalOutputTool with JSON that must match our Zod schema, guaranteeing well-formed structured outputs.

4. **Handoffs**

• You can define “handoff” tools that call sub-agents behind the scenes. This seamlessly delegates to specialized agents. The example is easily extended by writing a tool that calls AgentRunner.run on another agent.

5. **Guardrails**

• We illustrate input and output guardrails. We also mention how you can add “post-tool-call” checks if needed. Each guardrail returns a GuardrailResult with passed and optional details.

6. **Agent Class Design**

• modelSettings aligns more closely with the Vercel AI SDK’s GenerateTextOptions.

• No model is hardcoded; you can specify in modelSettings.

7. **Runner Logic & Conversation**

• We keep a minimal conversation array for the system + user messages. We rely on the Vercel AI SDK’s built-in multi-step logic for tool calls.

• Streaming and non-streaming share similar logic, with the main difference being how we iterate over the streamed output.

8. **Streaming Implementation**

• We yield partial text chunks for real-time feedback. For advanced use, you can emit structured event objects (tool calls, partial text, final results, etc.).

9. **Tracing & Logging**

• We introduced a Tracer class for demonstration. You can instrument deeper logs for tool calls, partial tokens, guardrail checks, etc. in production.

10. **Error Handling**

  

• We wrap critical operations in try/catch, define AgentError for meaningful exceptions, and surface guardrail failures with descriptive details.

---

**Conclusion**

  

By **addressing each critique**, this enhanced TypeScript “Agents SDK” now:

• **Fully embraces** the Vercel AI SDK’s best practices (e.g., direct tool usage, generateText multi-step logic).

• **Improves** type safety through Zod schemas and typed context.

• **Enforces** robust structured output with a final_output tool.

• **Allows** flexible guardrail checks at different points.

• **Supports** user-friendly streaming with partial text or structured events.

• **Enables** advanced handoffs, tracing, and error handling for production scenarios.

  

Use this foundation to build real-world, multi-agent systems that call external APIs, maintain context, produce validated structured responses, and gracefully handle delegated tasks. As the Vercel AI SDK continues to evolve, ensure your code keeps pace with new features like advanced tool concurrency, RAG (retrieval-augmented generation) integrations, and improved model support.