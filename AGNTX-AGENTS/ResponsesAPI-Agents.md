## Additional Responses API Documentation

This section includes additional documentation from the OpenAI Responses API and Vercel AI SDK that was not covered in the previous sections.

### OpenAI Responses API

#### Create a model response

**POST** `https://api.openai.com/v1/responses`

Creates a model response. Provide text or image inputs to generate text or JSON outputs. Have the model call your own custom code or use built-in tools like web search or file search to use your own data as input for the model's response.

**Request body:**

*   **input** (string or array, required): Text, image, or file inputs to the model, used to generate a response.
*   **model** (string, required): Model ID used to generate the response, like `gpt-4o` or `o1`.
*   **include** (array or null, optional): Specify additional output data to include in the model response.
*   **instructions** (string or null, optional): Inserts a system (or developer) message as the first item in the model's context.
*   **max\_output\_tokens** (integer or null, optional): An upper bound for the number of tokens that can be generated for a response.
*   **metadata** (map, optional): Set of 16 key-value pairs that can be attached to an object.
*   **parallel\_tool\_calls** (boolean or null, optional): Whether to allow the model to run tool calls in parallel. Defaults to true.
*   **previous\_response\_id** (string or null, optional): The unique ID of the previous response to the model.
*   **reasoning** (object or null, optional): Configuration options for reasoning models.
*   **store** (boolean or null, optional): Whether to store the generated model response for later retrieval via API. Defaults to true.
*   **stream** (boolean or null, optional): If set to true, the model response data will be streamed to the client as it is generated.
*   **temperature** (number or null, optional): What sampling temperature to use, between 0 and 2.
*   **text** (object, optional): Configuration options for a text response from the model.
*   **tool\_choice** (string or object, optional): How the model should select which tool (or tools) to use when generating a response.
*   **tools** (array, optional): An array of tools the model may call while generating a response.
*   **top\_p** (number or null, optional): An alternative to sampling with temperature, called nucleus sampling.
*   **truncation** (string or null, optional): The truncation strategy to use for the model response.
*   **user** (string, optional): A unique identifier representing your end-user.

#### Get a model response

**GET** `https://api.openai.com/v1/responses/{response_id}`

Retrieves a model response with the given ID.

#### Delete a model response

**DELETE** `https://api.openai.com/v1/responses/{response_id}`

Deletes a model response with the given ID.

#### List input items

**GET** `https://api.openai.com/v1/responses/{response_id}/input_items`

Returns a list of input items for a given response.

### Vercel AI SDK

#### Call Tools in Multiple Steps
Models call tools to gather information or perform actions that are not directly available to the model. When tool results are available, the model can use them to generate another response.

You can enable multi-step tool calls in generateText by setting the maxSteps option to a number greater than 1. This option specifies the maximum number of steps (i.e., LLM calls) that can be made to prevent infinite loops.

```typescript
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const { text } = await generateText({
  model: openai('gpt-4-turbo'),
  maxSteps: 5,
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      parameters: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }: { location: string }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
  },
  prompt: 'What is the weather in San Francisco?',
});
```

#### Call Tools in Parallel
Some language models support calling tools in parallel. This is particularly useful when multiple tools are independent of each other and can be executed in parallel during the same generation step.

```typescript
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const result = await generateText({
  model: openai('gpt-4-turbo'),
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      parameters: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }: { location: string }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
    cityAttractions: tool({
      parameters: z.object({ city: z.string() }),
      execute: async ({ city }: { city: string }) => {
        if (city === 'San Francisco') {
          return {
            attractions: [
              'Golden Gate Bridge',
              'Alcatraz Island',
              "Fisherman's Wharf",
            ],
          };
        } else {
          return { attractions: [] };
        }
      },
    }),
  },
  prompt:
    'What is the weather in San Francisco and what attractions should I visit?',
});

console.log(result);
```

#### Record Final Object after Streaming Object

When you're streaming structured data, you may want to record the final object for logging or other purposes. You can use the onFinish callback or the object promise.

#### Generate Text with Chat Prompt
A chat completion allows you to generate text based on a series of messages.

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

const result = await generateText({
  model: openai('gpt-3.5-turbo'),
  maxTokens: 1024,
  system: 'You are a helpful chatbot.',
  messages: [
    {
      role: 'user',
      content: 'Hello!',
    },
    {
      role: 'assistant',
      content: 'Hello! How can I help you today?',
    },
    {
      role: 'user',
      content: 'I need help with my computer.',
    },
  ],
});

console.log(result.text);

```

#### Generate Object with a Reasoning Model
Reasoning models, like DeepSeek's R1, are gaining popularity due to their ability to understand and generate better responses to complex queries than non-reasoning models. You may want to use these models to generate structured data. However, most (like R1 and OpenAI's o1) do not support tool-calling or structured outputs.

One solution is to pass the output from a reasoning model through a smaller model that can output structured data (like gpt-4o-mini).

#### Record Token Usage After Streaming Object
When you're streaming structured data with streamObject, you may want to record the token usage for billing purposes. You can use the onFinish callback or the usage promise.

#### Retrieval Augmented Generation
Retrieval Augmented Generation (RAG) is a technique that enhances the capabilities of language models by providing them with relevant information from external sources during the generation process.

#### Call Tools with Image Prompt
Some language models that support vision capabilities accept images as part of the prompt.

### Vercel AI SDK - Agents

When building AI applications, you often need **systems that can understand context and take meaningful actions**. When building these systems, the key consideration is finding the right balance between flexibility and control. Let's explore different approaches and patterns for building these systems, with a focus on helping you match capabilities to your needs.

#### Building Blocks

When building AI systems, you can combine these fundamental components:

##### Single-Step LLM Generation

The basic building block - one call to an LLM to get a response. Useful for straightforward tasks like classification or text generation.

##### Tool Usage

Enhanced capabilities through tools (like calculators, APIs, or databases) that the LLM can use to accomplish tasks. Tools provide a controlled way to extend what the LLM can do.

When solving complex problems, **an LLM can make multiple tool calls across multiple steps without you explicity specifying the order** \- for example, looking up information in a database, using that to make calculations, and then storing results. The AI SDK makes this [multi-step tool usage](https://sdk.vercel.ai/docs/foundations/agents#multi-step-tool-usage) straightforward through the `maxSteps` parameter.

##### Multi-Agent Systems

Multiple LLMs working together, each specialized for different aspects of a complex task. This enables sophisticated behaviors while keeping individual components focused.

#### Patterns

These building blocks can be combined with workflow patterns that help manage complexity:

*   **Sequential Processing (Chains)** - Steps executed in order
*   **Parallel Processing** - Independent tasks run simultaneously
*   **Evaluation/Feedback Loops** - Results checked and improved iteratively
*   **Orchestrator-Worker** - Coordinating multiple components
*   **Routing** - Directing work based on context

#### Choosing Your Approach

The key factors to consider:

*   **Flexibility vs Control** - How much freedom does the LLM need vs how tightly must you constrain its actions?
*   **Error Tolerance** - What are the consequences of mistakes in your use case?
*   **Cost Considerations** - More complex systems typically mean more LLM calls and higher costs
*   **Maintenance** - Simpler architectures are easier to debug and modify

**Start with the simplest approach that meets your needs**. Add complexity only when required by:

1.  Breaking down tasks into clear steps
2.  Adding tools for specific capabilities
3.  Implementing feedback loops for quality control
4.  Introducing multiple agents for complex workflows

Let's look at examples of these patterns in action.

#### Patterns with Examples

The following patterns, adapted from [Anthropic's guide on building effective agents](https://www.anthropic.com/research/building-effective-agents), serve as building blocks that can be combined to create comprehensive workflows. Each pattern addresses specific aspects of task execution, and by combining them thoughtfully, you can build reliable solutions for complex problems.

##### Sequential Processing (Chains)

The simplest workflow pattern executes steps in a predefined order. Each step's output becomes input for the next step, creating a clear chain of operations. This pattern is ideal for tasks with well-defined sequences, like content generation pipelines or data transformation processes.

```typescript
import { openai } from '@ai-sdk/openai';

import { generateText, generateObject } from 'ai';

import { z } from 'zod';

async function generateMarketingCopy(input: string) {

  const model = openai('gpt-4o');

  // First step: Generate marketing copy

  const { text: copy } = await generateText({

    model,

    prompt: `Write persuasive marketing copy for: ${input}. Focus on benefits and emotional appeal.`,

  });

  // Perform quality check on copy

  const { object: qualityMetrics } = await generateObject({

    model,

    schema: z.object({

      hasCallToAction: z.boolean(),

      emotionalAppeal: z.number().min(1).max(10),

      clarity: z.number().min(1).max(10),

    }),

    prompt: `Evaluate this marketing copy for:

    1. Presence of call to action (true/false)

    2. Emotional appeal (1-10)

    3. Clarity (1-10)

    Copy to evaluate: ${copy}`,

  });

  // If quality check fails, regenerate with more specific instructions

  if (

    !qualityMetrics.hasCallToAction ||

    qualityMetrics.emotionalAppeal < 7 ||

    qualityMetrics.clarity < 7

  ) {

    const { text: improvedCopy } = await generateText({

      model,

      prompt: `Rewrite this marketing copy with:

      ${!qualityMetrics.hasCallToAction ? '- A clear call to action' : ''}

      ${qualityMetrics.emotionalAppeal < 7 ? '- Stronger emotional appeal' : ''}

      ${qualityMetrics.clarity < 7 ? '- Improved clarity and directness' : ''}

      Original copy: ${copy}`,

    });

    return { copy: improvedCopy, qualityMetrics };

  }

  return { copy, qualityMetrics };

}
```

##### Routing

This pattern allows the model to make decisions about which path to take through a workflow based on context and intermediate results. The model acts as an intelligent router, directing the flow of execution between different branches of your workflow. This is particularly useful when handling varied inputs that require different processing approaches. In the example below, the results of the first LLM call change the properties of the second LLM call like model size and system prompt.

```typescript
import { openai } from '@ai-sdk/openai';

import { generateObject, generateText } from 'ai';

import { z } from 'zod';

async function handleCustomerQuery(query: string) {

  const model = openai('gpt-4o');

  // First step: Classify the query type

  const { object: classification } = await generateObject({

    model,

    schema: z.object({

      reasoning: z.string(),

      type: z.enum(['general', 'refund', 'technical']),

      complexity: z.enum(['simple', 'complex']),

    }),

    prompt: `Classify this customer query:

    ${query}

    Determine:

    1. Query type (general, refund, or technical)

    2. Complexity (simple or complex)

    3. Brief reasoning for classification`,

  });

  // Route based on classification

  // Set model and system prompt based on query type and complexity

  const { text: response } = await generateText({

    model:

      classification.complexity === 'simple'

        ? openai('gpt-4o-mini')

        : openai('o3-mini'),

    system: {

      general:

        'You are an expert customer service agent handling general inquiries.',

      refund:

        'You are a customer service agent specializing in refund requests. Follow company policy and collect necessary information.',

      technical:

        'You are a technical support specialist with deep product knowledge. Focus on clear step-by-step troubleshooting.',

    }[classification.type],

    prompt: query,

  });

  return { response, classification };

}
```

##### Parallel Processing

Some tasks can be broken down into independent subtasks that can be executed simultaneously. This pattern takes advantage of parallel execution to improve efficiency while maintaining the benefits of structured workflows. For example, analyzing multiple documents or processing different aspects of a single input concurrently (like code review).

```typescript
import { openai } from '@ai-sdk/openai';

import { generateText, generateObject } from 'ai';

import { z } from 'zod';

// Example: Parallel code review with multiple specialized reviewers

async function parallelCodeReview(code: string) {

  const model = openai('gpt-4o');

  // Run parallel reviews

  const [securityReview, performanceReview, maintainabilityReview] =

    await Promise.all([\

      generateObject({\

        model,\

        system:\

          'You are an expert in code security. Focus on identifying security vulnerabilities, injection risks, and authentication issues.',\

        schema: z.object({\

          vulnerabilities: z.array(z.string()),\

          riskLevel: z.enum(['low', 'medium', 'high']),\

          suggestions: z.array(z.string()),\

        }),\

        prompt: `Review this code:\

      ${code}`,\

      }),\

      generateObject({\
\
        model,\
\
        system:\
\
          'You are an expert in code performance. Focus on identifying performance bottlenecks, memory leaks, and optimization opportunities.',\
\
        schema: z.object({\
\
          issues: z.array(z.string()),\
\
          impact: z.enum(['low', 'medium', 'high']),\
\
          optimizations: z.array(z.string()),\
\
        }),\
\
        prompt: `Review this code:\
\
      ${code}`,\
\
      }),\
\
      generateObject({\
\
        model,\
\
        system:\
\
          'You are an expert in code quality. Focus on code structure, readability, and adherence to best practices.',\
\
        schema: z.object({\
\
          concerns: z.array(z.string()),\
\
          qualityScore: z.number().min(1).max(10),\
\
          recommendations: z.array(z.string()),\
\
        }),\
\
        prompt: `Review this code:\
\
      ${code}`,\
\
      }),\
\
    ]);

  const reviews = [\
\
    { ...securityReview.object, type: 'security' },\
\
    { ...performanceReview.object, type: 'performance' },\
\
    { ...maintainabilityReview.object, type: 'maintainability' },\
\
  ];

  // Aggregate results using another model instance

  const { text: summary } = await generateText({

    model,

    system: 'You are a technical lead summarizing multiple code reviews.',

    prompt: `Synthesize these code review results into a concise summary with key actions:

    ${JSON.stringify(reviews, null, 2)}`,

  });

  return { reviews, summary };

}
```

##### Orchestrator-Worker

In this pattern, a primary model (orchestrator) coordinates the execution of specialized workers. Each worker is optimized for a specific subtask, while the orchestrator maintains overall context and ensures coherent results. This pattern excels at complex tasks requiring different types of expertise or processing.

```typescript
import { openai } from '@ai-sdk/openai';

import { generateObject } from 'ai';

import { z } from 'zod';

async function implementFeature(featureRequest: string) {

  // Orchestrator: Plan the implementation

  const { object: implementationPlan } = await generateObject({

    model: openai('o3-mini'),

    schema: z.object({

      files: z.array(

        z.object({

          purpose: z.string(),

          filePath: z.string(),

          changeType: z.enum(['create', 'modify', 'delete']),

        }),

      ),

      estimatedComplexity: z.enum(['low', 'medium', 'high']),

    }),

    system:

      'You are a senior software architect planning feature implementations.',

    prompt: `Analyze this feature request and create an implementation plan:

    ${featureRequest}`,

  });

  // Workers: Execute the planned changes

  const fileChanges = await Promise.all(

    implementationPlan.files.map(async file => {

      // Each worker is specialized for the type of change

      const workerSystemPrompt = {

        create:

          'You are an expert at implementing new files following best practices and project patterns.',

        modify:

          'You are an expert at modifying existing code while maintaining consistency and avoiding regressions.',

        delete:

          'You are an expert at safely removing code while ensuring no breaking changes.',

      }[file.changeType];

      const { object: change } = await generateObject({

        model: openai('gpt-4o'),

        schema: z.object({

          explanation: z.string(),

          code: z.string(),

        }),

        system: workerSystemPrompt,

        prompt: `Implement the changes for ${file.filePath} to support:

        ${file.purpose}

        Consider the overall feature context:

        ${featureRequest}`,

      });

      return {

        file,

        implementation: change,

      };

    }),

  );

  return {

    plan: implementationPlan,

    changes: fileChanges,

  };

}
```

##### Evaluator-Optimizer

This pattern introduces quality control into workflows by having dedicated evaluation steps that assess intermediate results. Based on the evaluation, the workflow can either proceed, retry with adjusted parameters, or take corrective action. This creates more robust workflows capable of self-improvement and error recovery.

```typescript
import { openai } from '@ai-sdk/openai';

import { generateText, generateObject } from 'ai';

import { z } from 'zod';

async function translateWithFeedback(text: string, targetLanguage: string) {

  let currentTranslation = '';

  let iterations = 0;

  const MAX_ITERATIONS = 3;

  // Initial translation

  const { text: translation } = await generateText({

    model: openai('gpt-4o-mini'), // use small model for first attempt

    system: 'You are an expert literary translator.',

    prompt: `Translate this text to ${targetLanguage}, preserving tone and cultural nuances:

    ${text}`,

  });

  currentTranslation = translation;

  // Evaluation-optimization loop

  while (iterations < MAX_ITERATIONS) {

    // Evaluate current translation

    const { object: evaluation } = await generateObject({

      model: openai('gpt-4o'), // use a larger model to evaluate

      schema: z.object({

        qualityScore: z.number().min(1).max(10),

        preservesTone: z.boolean(),

        preservesNuance: z.boolean(),

        culturallyAccurate: z.boolean(),

        specificIssues: z.array(z.string()),

        improvementSuggestions: z.array(z.string()),

      }),

      system: 'You are an expert in evaluating literary translations.',

      prompt: `Evaluate this translation:

      Original: ${text}

      Translation: ${currentTranslation}

      Consider:

      1. Overall quality

      2. Preservation of tone

      3. Preservation of nuance

      4. Cultural accuracy`,

    });

    // Check if quality meets threshold

    if (

      evaluation.qualityScore >= 8 &&

      evaluation.preservesTone &&

      evaluation.preservesNuance &&

      evaluation.culturallyAccurate

    ) {

      break;

    }

    // Generate improved translation based on feedback

    const { text: improvedTranslation } = await generateText({

      model: openai('gpt-4o'), // use a larger model

      system: 'You are an expert literary translator.',

      prompt: `Improve this translation based on the following feedback:

      ${evaluation.specificIssues.join('\n')}

      ${evaluation.improvementSuggestions.join('\n')}

      Original: ${text}

      Current Translation: ${currentTranslation}`,

    });

    currentTranslation = improvedTranslation;

    iterations++;

  }

  return {

    finalTranslation: currentTranslation,

    iterationsRequired: iterations,

  };

}
```

### Multi-Step Tool Usage

If your use case involves solving problems where the solution path is poorly defined or too complex to map out as a workflow in advance, you may want to provide the LLM with a set of lower-level tools and allow it to break down the task into small pieces that it can solve on its own iteratively, without discrete instructions. To implement this kind of agentic pattern, you need to call an LLM in a loop until a task is complete. The AI SDK makes this simple with the `maxSteps` parameter.

With `maxSteps`, the AI SDK automatically triggers an additional request to the model after every tool result (each request is considered a "step"). If the model does not generate a tool call or the `maxSteps` threshold has been met, the generation is complete.

`maxSteps` can be used with both `generateText` and `streamText`

#### Using maxSteps

This example demonstrates how to create an agent that solves math problems.
It has a calculator tool (using [math.js](https://mathjs.org/)) that it can call to evaluate mathematical expressions.

```typescript
import { openai } from '@ai-sdk/openai';

import { generateText, tool } from 'ai';

import * as mathjs from 'mathjs';

import { z } from 'zod';

const { text: answer } = await generateText({

  model: openai('gpt-4o-2024-08-06', { structuredOutputs: true }),

  tools: {

    calculate: tool({

      description:

        'A tool for evaluating mathematical expressions. ' +

        'Example expressions: ' +

        "'1.2 * (2 + 4.5)', '12.7 cm to inch', 'sin(45 deg) ^ 2'.",

      parameters: z.object({ expression: z.string() }),

      execute: async ({ expression }) => mathjs.evaluate(expression),

    }),

  },

  maxSteps: 10,

  system:

    'You are solving math problems. ' +

    'Reason step by step. ' +

    'Use the calculator when necessary. ' +

    'When you give the final answer, ' +

    'provide an explanation for how you arrived at it.',

  prompt:

    'A taxi driver earns $9461 per 1-hour work. ' +

    'If he works 12 hours a day and in 1 hour ' +

    'he uses 12 liters of petrol with a price  of $134 for 1-liter. ' +

    'How much money does he earn in one day?',

});

console.log(`ANSWER: ${answer}`);
```

#### Structured Answers

When building an agent for tasks like mathematical analysis or report generation, it's often useful to have the agent's final output structured in a consistent format that your application can process. You can use an **answer tool** and the `toolChoice: 'required'` setting to force the LLM to answer with a structured output that matches the schema of the answer tool. The answer tool has no `execute` function, so invoking it will terminate the agent.

```typescript
import { openai } from '@ai-sdk/openai';

import { generateText, tool } from 'ai';

import 'dotenv/config';

import { z } from 'zod';

const { toolCalls } = await generateText({

  model: openai('gpt-4o-2024-08-06', { structuredOutputs: true }),

  tools: {

    calculate: tool({

      description:

        'A tool for evaluating mathematical expressions. Example expressions: ' +

        "'1.2 * (2 + 4.5)', '12.7 cm to inch', 'sin(45 deg) ^ 2'.",

      parameters: z.object({ expression: z.string() }),

      execute: async ({ expression }) => mathjs.evaluate(expression),

    }),

    // answer tool: the LLM will provide a structured answer

    answer: tool({

      description: 'A tool for providing the final answer.',

      parameters: z.object({

        steps: z.array(

          z.object({

            calculation: z.string(),

            reasoning: z.string(),

          }),

        ),

        answer: z.string(),

      }),

      // no execute function - invoking it will terminate the agent

    }),

  },

  toolChoice: 'required',

  maxSteps: 10,

  system:

    'You are solving math problems. ' +

    'Reason step by step. ' +

    'Use the calculator when necessary. ' +

    'The calculator can only do simple additions, subtractions, multiplications, and divisions. ' +

    'When you give the final answer, provide an explanation for how you got it.',

  prompt:

    'A taxi driver earns $9461 per 1-hour work. ' +

    'If he works 12 hours a day and in 1 hour he uses 14-liters petrol with price $134 for 1-liter. ' +

    'How much money does he earn in one day?',

});

console.log(`FINAL TOOL CALLS: ${JSON.stringify(toolCalls, null, 2)}`);
```

You can also use the
[`experimental_output`](https://sdk.vercel.ai/docs/ai-sdk-core/generating-structured-data#structured-output-with-generatetext)
setting for `generateText` to generate structured outputs.

#### Accessing all steps

Calling `generateText` with `maxSteps` can result in several calls to the LLM (steps).
You can access information from all steps by using the `steps` property of the response.

```typescript
import { generateText } from 'ai';

const { steps } = await generateText({

  model: openai('gpt-4o'),

  maxSteps: 10,

  // ...

});

// extract all tool calls from the steps:

const allToolCalls = steps.flatMap(step => step.toolCalls);
```

#### Getting notified on each completed step

You can use the `onStepFinish` callback to get notified on each completed step.
It is triggered when a step is finished,
i.e. all text deltas, tool calls, and tool results for the step are available.

```typescript
import { generateText } from 'ai';

const result = await generateText({

  model: yourModel,

  maxSteps: 10,

  onStepFinish({ text, toolCalls, toolResults, finishReason, usage }) {

    // your own logic, e.g. for saving the chat history or recording usage

  },

  // ...

});
```

### OpenAI Assistants API

#### Key Concepts

*   **Agent:** An LLM configured with instructions, tools, handoffs, guardrails, etc.
*   **Tool:** Functions the agent can call for external help (e.g., APIs, calculations, file access).
*   **Context:** A (mutable) object you create and pass along, storing state or shared resources.
*   **Output Types:** Allows you to specify structured final outputs (or default to free-form text).
*   **Handoffs:** Mechanism for delegating or switching the conversation to a different agent.
*   **Streaming:** Emits partial/delta output events as the agent thinks or calls tools (useful for real-time UIs).
*   **Tracing:** Automatically captures a detailed trace of each “agentic run” for debugging, analytics, or record-keeping.
*   **Guardrails:** Validate inputs or outputs, check policy, or halt execution if something is off-limits.

#### Agents

Agents encapsulate Large Language Models (LLMs) along with all necessary configuration, such as system prompts, available tools, and handoff targets. They can also include guardrails for input or output validation, plus advanced model settings.

**Core Idea**

An **Agent** is essentially an LLM with:

*   **Instructions** (like system prompts or dynamic instructions)
*   **Tools** it can call
*   **Handoffs** (delegation to other agents)
*   **Optional guardrails** for inputs or outputs
*   **Additional settings** (e.g. model parameters, output type, hooks

Responses API Firecrawled
Additional Responses API Documentation
This section includes additional documentation from the OpenAI Responses API and Vercel AI
SDK that was not covered in the previous sections.
OpenAI Responses API
Create a model response
POST https://api.openai.com/v1/responses
Creates a model response. Provide text or image inputs to generate text or JSON outputs.
Have the model call your own custom code or use built-in tools like web search or file search to
use your own data as input for the model's response.
Request body:
input (string or array, required): Text, image, or file inputs to the model, used to generate a
response.
model (string, required): Model ID used to generate the response, like gpt-4o or o1.
include (array or null, optional): Specify additional output data to include in the model
response.
instructions (string or null, optional): Inserts a system (or developer) message as the first
item in the model's context.
max_output_tokens (integer or null, optional): An upper bound for the number of tokens
that can be generated for a response.
metadata (map, optional): Set of 16 key-value pairs that can be attached to an object.
parallel_tool_calls (boolean or null, optional): Whether to allow the model to run tool calls
in parallel. Defaults to true.
previous_response_id (string or null, optional): The unique ID of the previous response to
the model.
reasoning (object or null, optional): Configuration options for reasoning models.
store (boolean or null, optional): Whether to store the generated model response for later
retrieval via API. Defaults to true.
stream (boolean or null, optional): If set to true, the model response data will be streamed
to the client as it is generated.
temperature (number or null, optional): What sampling temperature to use, between 0 and
2.
text (object, optional): Configuration options for a text response from the model.
tool_choice (string or object, optional): How the model should select which tool (or tools) to
use when generating a response.
tools (array, optional): An array of tools the model may call while generating a response.
top_p (number or null, optional): An alternative to sampling with temperature, called
nucleus sampling.
truncation (string or null, optional): The truncation strategy to use for the model response.
user (string, optional): A unique identifier representing your end-user.
Get a model response
GET https://api.openai.com/v1/responses/{response_id}
Retrieves a model response with the given ID.
Delete a model response
DELETE https://api.openai.com/v1/responses/{response_id}
Deletes a model response with the given ID.
List input items
GET https://api.openai.com/v1/responses/{response_id}/input_items
Returns a list of input items for a given response.
Vercel AI SDK
Call Tools in Multiple Steps
Models call tools to gather information or perform actions that are not directly available to the
model. When tool results are available, the model can use them to generate another response.
You can enable multi-step tool calls in generateText by setting the maxSteps option to a number
greater than 1. This option specifies the maximum number of steps (i.e., LLM calls) that can be
made to prevent infinite loops.
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';
const { text } = await generateText({
model: openai('gpt-4-turbo'),
maxSteps: 5,
tools: {
weather: tool({
description: 'Get the weather in a location',
parameters: z.object({
location: z.string().describe('The location to get the weather
for'),
}),
execute: async ({ location }: { location: string }) => ({
location,
temperature: 72 + Math.floor(Math.random() * 21) - 10,
}),
}),
},
prompt: 'What is the weather in San Francisco?',
});
Call Tools in Parallel
Some language models support calling tools in parallel. This is particularly useful when multiple
tools are independent of each other and can be executed in parallel during the same generation
step.
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';
const result = await generateText({
model: openai('gpt-4-turbo'),
tools: {
weather: tool({
description: 'Get the weather in a location',
parameters: z.object({
location: z.string().describe('The location to get the weather
for'),
}),
execute: async ({ location }: { location: string }) => ({
location,
temperature: 72 + Math.floor(Math.random() * 21) - 10,
}),
}),
cityAttractions: tool({
parameters: z.object({ city: z.string() }),
execute: async ({ city }: { city: string }) => {
if (city === 'San Francisco') {
return {
attractions: [
'Golden Gate Bridge',
'Alcatraz Island',
"Fisherman's Wharf",
],
};
} else {
return { attractions: [] };
}
},
}),
},
prompt:
visit?',
});
'What is the weather in San Francisco and what attractions should I
console.log(result);
Record Final Object after Streaming Object
When you're streaming structured data, you may want to record the final object for logging or
other purposes. You can use the onFinish callback or the object promise.
Generate Text with Chat Prompt
A chat completion allows you to generate text based on a series of messages.
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
const result = await generateText({
model: openai('gpt-3.5-turbo'),
maxTokens: 1024,
system: 'You are a helpful chatbot.',
messages: [
{
role: 'user',
content: 'Hello!',
},
{
role: 'assistant',
content: 'Hello! How can I help you today?',
},
{
role: 'user',
content: 'I need help with my computer.',
},
],
});
console.log(result.text);
Generate Object with a Reasoning Model
Reasoning models, like DeepSeek's R1, are gaining popularity due to their ability to understand
and generate better responses to complex queries than non-reasoning models. You may want
to use these models to generate structured data. However, most (like R1 and OpenAI's o1) do
not support tool-calling or structured outputs.
One solution is to pass the output from a reasoning model through a smaller model that can
output structured data (like gpt-4o-mini).
Record Token Usage After Streaming Object
When you're streaming structured data with streamObject, you may want to record the token
usage for billing purposes. You can use the onFinish callback or the usage promise.
Retrieval Augmented Generation
Retrieval Augmented Generation (RAG) is a technique that enhances the capabilities of
language models by providing them with relevant information from external sources during the
generation process.
Call Tools with Image Prompt
Some language models that support vision capabilities accept images as part of the prompt.
Mapping Vercel AI SDK and OpenAI Responses
API Integrations
This document maps the required VERCel AI SDK and Responses API integrations mentioned
in the 'ResponsesAPI/Master-Plan.md' for upgrading CUA-Browser with OpenManus
capabilities.
OpenAI Responses API Overview
The OpenAI Responses API provides a new way to build applications on OpenAI's platform.
Key features include:
Persistent Chat History: The API allows persisting chat history across requests, enabling
multi-turn conversations.
Built-in Tools: The API offers built-in tools like webSearch for grounding responses in real-
time web data and fileSearch for accessing information from uploaded files.
Computer Use Tool: This tool enables building agents that can interact with and operate
computers, facilitating tasks like web browsing and automation.
Function Calling: The API supports function calling, allowing the model to interact with
external systems and perform discrete tasks.
Vercel AI SDK Integration
The Vercel AI SDK is a TypeScript toolkit designed for building AI applications with LLMs. It
simplifies integration with the OpenAI Responses API by:
Abstracting Model Provider Differences: The SDK provides a unified API to call any LLM,
including the OpenAI Responses API.
Eliminating Boilerplate: It reduces the amount of code needed for common tasks like
building chatbots.
Supporting Rich Outputs: The SDK allows going beyond text outputs to generate
structured data and interactive components.
Example (Basic Usage):
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
const { text } = await generateText({
model: openai.responses('gpt-4o'),
prompt: 'Explain the concept of quantum entanglement.',
});
Tool Implementations
The Master-Plan.md outlines several tools to be implemented using the OpenAI Responses
API and the Vercel AI SDK.
Browser Control (Computer-Use Tool)
This tool leverages OpenAI's Computer Use Agent (CUA) model and Browserbase for browser
automation.
Integration: The existing BrowserbaseBrowser class from CUA-Browser (in
app/api/agent/browserbase.ts) will be adapted.
Mechanism: The agent will output a computer_call with actions like "click(x,y)" or
"type(...)". These actions will be captured and executed via the Browserbase API.
Observation: After each action, the updated screenshot or page text will be sent back to
the model as the function result.
Vercel AI SDK: The openai.responses provider will be used, potentially with a special
loop to handle the CUA model's multi-call nature.
Web Search Tool
This tool allows the agent to query the web for information.
Function Definition:
const searchTool = {
name: "web_search",
description: "Search the web for information and return a list of result
URLs.",
parameters: {
type: "object",
properties: {
query: { type: "string", description: "Search query term(s)" },
num_results: { type: "integer", description: "Max number of results",
default: 5 }
},
required: ["query"]
},
execute: async ({ query, num_results }) => {
// Call an external search API or service (e.g., Brave Search API)
const results = await performSearch(query, num_results);
return results;
}
};
Vercel AI SDK: The openai.responses provider will be used, and the web_search tool
will be registered in the functions array. The performSearch function will need to be
implemented to call an external search API (e.g., Brave Search API). The Vercel AI SDK
also provides built in support for the web search tool:
import { openai } from '@ai-sdk/openai';
import { generateText } from 'ai';
const result = await generateText({
model: openai.responses('gpt-4o-mini'),
prompt: 'What happened in San Francisco last week?',
tools: {
web_search_preview: openai.tools.webSearchPreview(),
},
});
console.log(result.text);
console.log(result.sources);
Code Execution Tool
This tool enables the agent to execute code for computations and data processing.
Function Definition: execute_code with an input schema for the code string (and
potentially a language flag).
Implementation: For simplicity, JavaScript execution will be supported initially using
Node.js's vm module for sandboxing.
Security: Strict security measures will be implemented (no unlimited file system access, no
infinite loops).
Vercel AI SDK: The openai.responses provider will be used, and the execute_code tool
will be registered in the functions array.
Example:
import { NodeVM } from 'vm2';
const vm = new NodeVM({ timeout: 5000, sandbox: {} });
try {
const result = vm.run(userCode, 'sandbox.js');
return String(result);
} catch(e) {
return `Error: ${e.message}`
;
}
File Management Tools
These tools enable the agent to save and read files.
file_save : Uses Node.js's fs.writeFile to save data to the /tmp directory on Vercel.
file_read (optional): Uses fs.readFile to read content from files.
Vercel AI SDK: The openai.responses provider will be used, and the file_save and
file_read tools will be registered in the functions array.
Agent Workflow and Long-Horizon Task Handling
The agent will follow a multi-phase architecture:
1. Planning: For complex tasks, the agent will generate a plan (list of steps) using a planning
prompt template.
2. Iterative Tool-Use Loop:
Select Step & Tool: The agent identifies the current step and chooses an appropriate
tool.
Execute Tool: The chosen tool function runs.
Observe & Analyze: The LLM processes the tool's result.
Update Plan/State: The agent updates the plan's status.
Loop Continuation: The cycle repeats until all steps are done or the agent has enough
information.
3. Long Report Generation: After gathering information, the agent composes the final output.
4. Iterative Coding: The agent can write, execute, and debug code iteratively.
Memory: The conversation history (including user requests, plan, and tool outputs) serves as
the agent's memory.
Error Handling: Robust error handling will be implemented for tool errors and unexpected
situations.
Streaming and UI Updates
The Vercel AI SDK's streaming capabilities will be used to provide real-time updates to the UI:
StreamingTextResponse : Stream tokens to the client as they arrive.
Interim Messages: Send interim messages like "(Agent is browsing...)" during tool usage.
Browser Viewport: Update the browser viewport (if present) with screenshots from
Browserbase during browser actions.
Migration from Completions API
The Vercel AI SDK provides a simple migration path:
Change the provider instance from openai(modelId) to openai.responses(modelId).
Move provider-specific options from the model provider instance to the providerOptions
object.
Example:
// Completions API
const { text } = await generateText({
model: openai('gpt-4o', { parallelToolCalls: false }),
prompt: 'Explain the concept of quantum entanglement.',
});
// Responses API
const { text } = await generateText({
model: openai.responses('gpt-4o'),
prompt: 'Explain the concept of quantum entanglement.',
providerOptions: {
openai: {
parallelToolCalls: false,
},
},
});
Function Calling
Function calling provides a powerful and flexible way for OpenAI models to interface with your
code or external services.
Defining Functions:
Functions are defined by their schema, which informs the model what it does and what input
arguments it expects. It comprises the following fields:
type: This should always be function
name: The function's name (e.g. get_weather)
description: Details on when and how to use the function
parameters: JSON schema defining the function's input arguments
strict: Whether to enforce strict mode for the function call
Example Function Schema:
{
"type": "function",
"function": {
"name": "get_weather",
"description": "Retrieves current weather for the given location.",
"parameters": {
"type": "object",
"properties": {
"location": {
"type": "string",
"description": "City and country e.g. Bogotá, Colombia"
},
"units": {
"type": "string",
"enum": [
"celsius",
"fahrenheit"
],
"description": "Units the temperature will be returned
in."
}
},
"required": [
"location",
"units"
],
"additionalProperties": false
},
"strict": true
}
}
Handling Function Calls:
1. Call model with functions defined: Along with your system and user messages.
2. Model decides to call function(s): Model returns the name and input arguments.
3. Execute function code: Parse the model's response and handle function calls.
4. Supply model with results: So it can incorporate them into its final response.
5. Model responds: Incorporating the result in its output.
Additional Configurations:
Tool Choice: Control how the model selects tools ( auto, required, or a specific
function).
Parallel Function Calling: Allow the model to run multiple tool calls in parallel.
Strict Mode: Enforce strict adherence to the function schema.
Streaming:
Streaming can be used to surface progress by showing which function is called as the model
fills its arguments, and even displaying the arguments in real time.
Additional Responses API Documentation
This section includes additional documentation from the OpenAI Responses API and Vercel AI
SDK that was not covered in the previous sections.
OpenAI Responses API
Create a model response
POST https://api.openai.com/v1/responses
Creates a model response. Provide text or image inputs to generate text or JSON outputs.
Have the model call your own custom code or use built-in tools like web search or file search to
use your own data as input for the model's response.
Request body:
input (string or array, required): Text, image, or file inputs to the model, used to generate a
response.
model (string, required): Model ID used to generate the response, like gpt-4o or o1.
include (array or null, optional): Specify additional output data to include in the model
response.
instructions (string or null, optional): Inserts a system (or developer) message as the first
item in the model's context.
max_output_tokens (integer or null, optional): An upper bound for the number of tokens
that can be generated for a response.
metadata (map, optional): Set of 16 key-value pairs that can be attached to an object.
parallel_tool_calls (boolean or null, optional): Whether to allow the model to run tool calls
in parallel. Defaults to true.
previous_response_id (string or null, optional): The unique ID of the previous response to
the model.
reasoning (object or null, optional): Configuration options for reasoning models.
store (boolean or null, optional): Whether to store the generated model response for later
retrieval via API. Defaults to true.
stream (boolean or null, optional): If set to true, the model response data will be streamed
to the client as it is generated.
temperature (number or null, optional): What sampling temperature to use, between 0 and
2.
text (object, optional): Configuration options for a text response from the model.
tool_choice (string or object, optional): How the model should select which tool (or tools) to
use when generating a response.
tools (array, optional): An array of tools the model may call while generating a response.
top_p (number or null, optional): An alternative to sampling with temperature, called
nucleus sampling.
truncation (string or null, optional): The truncation strategy to use for the model response.
user (string, optional): A unique identifier representing your end-user.
Get a model response
GET https://api.openai.com/v1/responses/{response_id}
Retrieves a model response with the given ID.
Delete a model response
DELETE https://api.openai.com/v1/responses/{response_id}
Deletes a model response with the given ID.
List input items
GET https://api.openai.com/v1/responses/{response_id}/input_items
Returns a list of input items for a given response.
Vercel AI SDK
Call Tools in Multiple Steps
Models call tools to gather information or perform actions that are not directly available to the
model. When tool results are available, the model can use them to generate another response.
You can enable multi-step tool calls in generateText by setting the maxSteps option to a number
greater than 1. This option specifies the maximum number of steps (i.e., LLM calls) that can be
made to prevent infinite loops.
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';
const { text } = await generateText({
model: openai('gpt-4-turbo'),
maxSteps: 5,
tools: {
weather: tool({
description: 'Get the weather in a location',
parameters: z.object({
location: z.string().describe('The location to get the weather
for'),
}),
execute: async ({ location }: { location: string }) => ({
location,
temperature: 72 + Math.floor(Math.random() * 21) - 10,
}),
}),
},
prompt: 'What is the weather in San Francisco?',
});
Call Tools in Parallel
Some language models support calling tools in parallel. This is particularly useful when multiple
tools are independent of each other and can be executed in parallel during the same generation
step.
import { generateText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';
const result = await generateText({
model: openai('gpt-4-turbo'),
tools: {
weather: tool({
description: 'Get the weather in a location',
parameters: z.object({
location: z.string().describe('The location to get the weather
for'),
}),
execute: async ({ location }: { location: string }) => ({
location,
temperature: 72 + Math.floor(Math.random() * 21) - 10,
}),
}),
cityAttractions: tool({
parameters: z.object({ city: z.string() }),
execute: async ({ city }: { city: string }) => {
if (city === 'San Francisco') {
return {
attractions: [
'Golden Gate Bridge',
'Alcatraz Island',
"Fisherman's Wharf",
],
};
} else {
return { attractions: [] };
}
},
}),
},
prompt:
visit?',
});
'What is the weather in San Francisco and what attractions should I
console.log(result);
Record Final Object after Streaming Object
When you're streaming structured data, you may want to record the final object for logging or
other purposes. You can use the onFinish callback or the object promise.
Generate Text with Chat Prompt
A chat completion allows you to generate text based on a series of messages.
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
const result = await generateText({
model: openai('gpt-3.5-turbo'),
maxTokens: 1024,
system: 'You are a helpful chatbot.',
messages: [
{
role: 'user',
content: 'Hello!',
},
{
role: 'assistant',
content: 'Hello! How can I help you today?',
},
{
role: 'user',
content: 'I need help with my computer.',
},
],
});
console.log(result.text);
Generate Object with a Reasoning Model
Reasoning models, like DeepSeek's R1, are gaining popularity due to their ability to understand
and generate better responses to complex queries than non-reasoning models. You may want
to use these models to generate structured data. However, most (like R1 and OpenAI's o1) do
not support tool-calling or structured outputs.
One solution is to pass the output from a reasoning model through a smaller model that can
output structured data (like gpt-4o-mini).
Record Token Usage After Streaming Object
When you're streaming structured data with streamObject, you may want to record the token
usage for billing purposes. You can use the onFinish callback or the usage promise.
Retrieval Augmented Generation
Retrieval Augmented Generation (RAG) is a technique that enhances the capabilities of
language models by providing them with relevant information from external sources during the
generation process.
Call Tools with Image Prompt
Some language models that support vision capabilities accept images as part of the prompt.