Agents SDK
==========

Learn how to build agents with the OpenAI.

Welcome to the **OpenAI Agents SDK**. This library makes it straightforward to build **agentic applications**—where a Large Language Model (LLM) can leverage additional context and tools, possibly hand off to other specialized agents, stream partial results, and keep a full trace of what happened.

Below is a thorough overview. By the end of this guide, you’ll have a strong understanding of core concepts, best practices, and how to assemble everything in a well-structured agent workflow.

Download and installation
-------------------------

Access the latest version of the Agents SDK [here on GitHub](https://github.com/openai/openai-agents-python).

* * *

Quick Links
-----------

*   [Key Concepts](#key-concepts)
*   [Agents](#agents)
*   [Tools](#tools)
*   [Context](#context)
*   [Output Types](#output-types)
*   [Handoffs](#handoffs)
*   [Streaming](#streaming)
*   [Tracing](#tracing)
*   [Guardrails](#guardrails)
*   [Putting It All Together](#putting-it-all-together)
*   [Best Practices](#best-practices)
*   [Next Steps](#next-steps)

* * *

Key Concepts
------------

The Agents SDK is designed around these core primitives:

|Component|Purpose|
|---|---|
|Agent|An LLM configured with instructions, tools, handoffs, guardrails, etc.|
|Tool|Functions the agent can call for external help (e.g., APIs, calculations, file access).|
|Context|A (mutable) object you create and pass along, storing state or shared resources.|
|Output Types|Allows you to specify structured final outputs (or default to free-form text).|
|Handoffs|Mechanism for delegating or switching the conversation to a different agent.|
|Streaming|Emits partial/delta output events as the agent thinks or calls tools (useful for real-time UIs).|
|Tracing|Automatically captures a detailed trace of each “agentic run” for debugging, analytics, or record-keeping.|
|Guardrails|Validate inputs or outputs, check policy, or halt execution if something is off-limits.|

Bringing these together, you can create powerful multi-step or conversational workflows, ensuring correct, verifiable results—and if needed, break down tasks or call specialized sub-agents with guardrails, streaming, or structured outputs.

* * *

Agents
------

Agents encapsulate Large Language Models (LLMs) along with all necessary configuration, such as system prompts, available tools, and handoff targets. They can also include guardrails for input or output validation, plus advanced model settings.

### Core Idea

An **Agent** is essentially an LLM with:

*   **Instructions** (like system prompts or dynamic instructions)
*   **Tools** it can call
*   **Handoffs** (delegation to other agents)
*   **Optional guardrails** for inputs or outputs
*   **Additional settings** (e.g. model parameters, output type, hooks)

When you invoke an agent with user input, it runs an **agent loop**:

1.  The agent sees the conversation so far (including user input, previous tool results, system messages).
2.  The LLM decides to:
    *   Produce a final answer (ending the loop),
    *   Call a tool (possibly multiple times in sequence),
    *   Or hand off to another agent.
3.  Once the agent produces a final answer (or hands off), the loop ends.

Example: Basic Agent

```python
from agents import Agent, AgentRunner

agent = Agent(
name="basic_agent",
instructions="You are a helpful agent."
)

result = await AgentRunner.run(agent, ["What's the capital of the USA?"])
print(result.agent_output)
# The capital of the United States is Washington, D.C.
```

### Agent Fields

*   **`name`**: Short identifier for the agent.
*   **`instructions`** or **`instructions_function`**:
    *   `instructions`: A static string describing how the agent should respond.
    *   `instructions_function`: A function returning a string (for dynamic instructions).
*   **`description`**: Explains the agent’s role (often shown if this agent is used as a sub-agent/tool).
*   **`tools`**: A list of external functions (Tools) the agent can call.
*   **`handoffs`**: A list of possible handoff targets or sub-agents.
*   **`model_settings`**: Parameters for LLM usage (model type, temperature, etc.).
*   **`output_type`**: By default, final output is a string. You can require structured output instead.
*   **`context_type`**: Sets a Python type for your context object (see [Context](#context)).

#### Creating or Cloning Agents

```python
from agents import Agent

# Basic
my_agent = Agent(
    name="my_agent",
    instructions="You are extremely friendly and helpful."
)

# Cloning with modifications
new_agent = my_agent.clone(
    name="my_new_agent",
    instructions="Now you're more formal."
)
```

### The Agent Loop

When you do:

```python
final_result = await AgentRunner.run(agent, ["some user input"])
```

Under the hood:

1.  The LLM sees the conversation (including user input, system messages, prior tool outputs).
2.  The LLM might produce a final user-facing message **or** call a tool **or** trigger a handoff.
3.  If it calls a tool, the result is fed back into the conversation. The process repeats until a final message or handoff is reached.
4.  If using a structured output type (e.g., a Pydantic model), the agent must call a special `final_output` tool to produce valid JSON. If the JSON is invalid, you’ll see a `ModelBehaviorError`.
5.  The framework enforces a `max_turns` safety measure to prevent infinite loops.

* * *

Tools
-----

Tools let agents invoke external functions (e.g., for accessing APIs, running code, performing searches). By exposing a Python function as a tool, the LLM can decide at runtime to call it with arguments, see results, and use that to inform the next step.

### Defining Tools

You define tools with the `@function_tool` decorator (or `function_tool(...)`):

Define a weather retrieval tool

```python
from agents.tool import function_tool

@function_tool
def get_weather(location: str, unit: str = "C") -> str:
  """
  Fetch the weather for a given location, returning a short description.
  """
  # Example logic
  return f"The weather in {location} is 22 degrees {unit}."
```

Attach it to an agent:

```python
agent.tools.append(get_weather)
```

*   Tools can be synchronous or asynchronous.
*   You can optionally take a typed context as the first argument (see next section).
*   The agent sees each tool by name and can call it with valid JSON arguments.

**Common Patterns**:

*   Tools for math, web requests, data lookups, specialized computations, etc.
*   Tools can log or store data in your system if needed.
*   If you have many tools, consider distinct instructions to guide the LLM on how to choose them.

* * *

Context
-------

Sometimes you need to keep session data, credentials, or ephemeral state across multiple tool calls or steps. The Agents SDK allows a **context** object to be passed into the run.

*   You create any Python class as context, or use a typed approach with `Agent(context_type=MyContext)`.
*   Tools can then access that context, read data, and update it.

Using Context for User Sessions

```python
from agents.run_context import AgentContextWrapper
from agents.tool import function_tool

class MyContext:
  def __init__(self, user_id: str):
      self.user_id = user_id
      self.seen_messages = []

@function_tool
def greet_user(context: AgentContextWrapper[MyContext], greeting: str) -> str:
  user_id = context.agent_context.user_id
  return f"Hello {user_id}, you said: {greeting}"

agent = Agent(
  name="my_agent_with_context",
  context_type=MyContext,
  tools=[greet_user],
)

my_ctx = MyContext(user_id="alice")
result = await AgentRunner.run(
  agent,
  input=["Hi agent!"],
  context=my_ctx,
)
```

Here, the `greet_user` tool can access `context.agent_context.user_id`. You can store state, counters, or other shared resources. This pattern is invaluable for multi-step reasoning or advanced logic within tool calls.

* * *

Output Types
------------

Agents can produce free-form text (the default) or structured outputs.

### Default (String) Output

By default, an agent’s final answer is simply the next text message produced without calling a tool. Once the agent returns that text, the run ends.

```python
agent = Agent(
    name="StringOutputAgent",
    instructions="Always provide final answers as text, never a tool."
)
out = await AgentRunner.run(agent, ["Summarize thermodynamics"])
print(out.agent_output)
# "Thermodynamics is the study of heat, energy, and work..."
```

### Structured Outputs

Setting `output_type` forces the agent to produce a JSON object that matches a Python (often Pydantic) schema. The LLM calls a special `final_output` tool, which validates the JSON.

Structured output with Pydantic

```python
from pydantic import BaseModel
from agents import Agent, AgentRunner

class WeatherAnswer(BaseModel):
  location: str
  temperature_c: float
  summary: str

agent = Agent(
  name="StructuredWeatherAgent",
  instructions="Use the final_output tool with WeatherAnswer schema.",
  output_type=WeatherAnswer,
)

out = await AgentRunner.run(agent, ["What's the temperature in Paris?"])
print(type(out.agent_output))
# <class '__main__.WeatherAnswer'>
print(out.agent_output.temperature_c)
# e.g. 22.0
```

If the LLM returns malformed JSON, the run errors out, ensuring your final data is guaranteed valid.

* * *

Handoffs
--------

Handoffs let one agent delegate a conversation to another agent. This is useful when you have multiple specialized agents (e.g., a “Spanish-only” agent vs. “English-only” agent) or domain-specific agents (tech support vs. billing).

### High-Level Flow

1.  A parent (or “triage”) agent can see multiple sub-agent handoff targets.
2.  The parent decides to invoke a “handoff tool” referencing a specific sub-agent.
3.  The SDK switches to that sub-agent for subsequent steps until it finishes or delegates again.

Handoff example

```python
from agents import Agent, AgentRunner, handoff

spanish_agent = Agent(name="spanish_agent", instructions="You only speak Spanish.")
english_agent = Agent(name="english_agent", instructions="You only speak English.")

triage_agent = Agent(
  name="triage_agent",
  instructions="Handoff to the appropriate agent based on language.",
  handoffs=[spanish_agent, english_agent],
)

out = await AgentRunner.run(triage_agent, ["Hola, ¿cómo estás?"])
# The conversation is handed off to spanish_agent.
# Final answer will be in Spanish.
```

Behind the scenes:

*   Each sub-agent is wrapped in a `handoff(...)` object, which exposes a tool like `"handoff_spanish_agent"`.
*   If the parent agent calls that tool, the SDK shifts the active agent to `spanish_agent`.

#### Handoff Input and Filters

You can define arguments for the new agent, or filter conversation history passed along. For example, limiting how much context the sub-agent sees or storing info in your own callback.

* * *

Streaming
---------

**Streaming** is used when you want partial or incremental output (like a real-time chat experience). Instead of `AgentRunner.run(...)`, call `AgentRunner.run_streamed(...)` to get an async iterator of events.

Streaming mode example

```python
from agents import Agent, AgentRunner

stream = AgentRunner.run_streamed(agent, ["Tell me a story"])

async for event in stream.stream_events():
  # event.delta often includes partial text or function call info
  print(event.delta, end="", flush=True)
print("\n--- done ---")
```

Events could include:

*   Partial text responses
*   Tool call argument deltas
*   Indications that a final answer was produced

The agent can still call multiple tools, produce partial messages, or do a handoff while streaming.

* * *

Tracing
-------

**Tracing** automatically records each step of an agent’s run: user messages, system prompts, tool calls, function call inputs and outputs, plus final responses. This is invaluable for debugging, analytics, or storing conversation transcripts.

*   By default, the SDK does `trace(name="agent_run")` whenever you call `AgentRunner.run(...)`.
*   You can customize or disable it with `AgentRunConfig` or environment variables like:
    *   `LLM_DEBUG=true`
    *   `AGENT_LIFECYCLE=true`
    *   `TOOL_DEBUG=true`

If you need to store or process traces yourself, implement a custom `TracingProcessor` and register it via `add_trace_processor` or `set_trace_processors([...])`.

Example configuration:

```python
from agents import AgentRunner, AgentRunConfig

config = AgentRunConfig(
    run_name="CustomerSupportFlow",
    tracing_disabled=False,
    trace_non_openai_generations=True,
)

result = await AgentRunner.run(my_agent, ["Hello?"], run_config=config)
```

* * *

Guardrails
----------

Guardrails let you enforce policies on agent inputs or outputs, such as blocking disallowed content or verifying data integrity. If a guardrail fails, the run can end immediately or raise an exception.

### Configuring Guardrails

```python
from agents.guardrails import CustomGuardrail

async def is_not_swearing(msgs, context) -> bool:
    content = " ".join(m["content"] for m in msgs if "content" in m)
    return "badword" not in content.lower()

my_guardrail = CustomGuardrail(
    guardrail_function=is_not_swearing,
    tripwire_config=lambda output: not output  # if 'False', raise error
)

agent = Agent(
    name="my_agent",
    input_guardrails=[my_guardrail]
)

out = await AgentRunner.run(agent, ["some text"])
# If "badword" is in input, GuardrailTripwireTriggered is raised.
```

You can also run LLM-based guardrails that apply an AI-driven policy check. Guardrails can be placed on input, output, or intermediate steps, depending on your usage.

* * *

Putting It All Together
-----------------------

Below is a conceptual snippet demonstrating multiple features: instructions, tools, handoffs, structured output, and run config. You can also apply streaming or tracing as desired.

Multi-agent workflow example

```python
from agents import Agent, AgentRunner, AgentRunConfig, handoff
from agents.tool import function_tool
from pydantic import BaseModel

class StructuredAnswer(BaseModel):
  decision: str
  reasoning: str

@function_tool
def get_forecast(city: str) -> str:
  return f"The forecast in {city} is sunny."

spanish_agent = Agent(name="spanish_agent", instructions="You only speak Spanish.")
english_agent = Agent(name="english_agent", instructions="You only speak English.")

triage_agent = Agent(
  name="triage_agent",
  instructions="Switch to Spanish if user uses Spanish, otherwise English.",
  tools=[get_forecast],
  handoffs=[handoff(spanish_agent), handoff(english_agent)],
  output_type=StructuredAnswer,
)

config = AgentRunConfig(run_name="multilingual_weather_run")
output = await AgentRunner.run(
  triage_agent, 
  ["Hola, ¿qué tiempo hace en Madrid?"], 
  run_config=config
)

print(output.agent_output)
# => StructuredAnswer(decision='Spanish agent used', reasoning='Detected Spanish phrase')
```

### Another Comprehensive Snippet

```python
from agents import Agent, AgentRunner, handoff
from agents.tool import function_tool
from pydantic import BaseModel

class WeatherResponse(BaseModel):
    location: str
    forecast: str

@function_tool
def get_forecast(city: str) -> str:
    return f"{city} is sunny."

spanish_agent = Agent(name="spanish_agent", instructions="Respond only in Spanish.")
english_agent = Agent(name="english_agent", instructions="Respond only in English.")

triage_agent = Agent(
    name="triage_agent",
    instructions="Use language detection to hand off appropriately.",
    tools=[get_forecast],
    handoffs=[handoff(spanish_agent), handoff(english_agent)],
    output_type=WeatherResponse,
)

config = AgentRunConfig(run_name="weather_check")
output = await AgentRunner.run(
    triage_agent,
    ["Hola, clima en Madrid?"],
    run_config=config
)

print(output.agent_output)
# WeatherResponse(location='Madrid', forecast='Madrid is sunny.')
```

* * *

Best Practices
--------------

*   **Document function schemas** carefully so the LLM knows how to call each tool.
*   Use strict schema checking (e.g., a Pydantic model) to avoid partial or malformed arguments.
*   Apply **guardrails** that make sense for your domain—like content filters, usage quotas, or data validation.
*   For complex multi-step flows, **trace** thoroughly. The debug logs or custom trace processors can be a lifesaver.
*   Consider employing handoffs for specialized sub-agents if your application covers multiple domains or languages.

* * *

Next Steps
----------

*   Check out our [examples folder](/examples/agent_patterns/) for reference use cases:
    *   **Chat Agent**: A typical QA or chat-based helper with some tools.
    *   **Multi-step reasoning**: Agents that iterate with multiple tools.
    *   **Handoff**: Triage to specialized sub-agents.
    *   **Guardrail** usage: Implementing custom policy checks.
    *   **Tracing** advanced usage: Fine-tuning trace storage or disabling it.
*   Read docstrings in the code for deeper details.

That’s the **OpenAI Agents SDK** in depth:

1.  **Agents** define LLM personalities and capabilities (tools, handoffs, guardrails).
2.  **Tools** provide external functionality for the LLM to call.
3.  **Context** shares data between steps.
4.  **Output Types** ensure you get structured or free-form outputs as needed.
5.  **Handoffs** allow delegation to specialized agents.
6.  **Streaming** provides incremental output events.
7.  **Tracing** records everything for debug/analytics.
8.  **Guardrails** keep everything within policy or usage limits.

**Happy building with the Agents SDK!**

Was this page useful?

Structured Outputs
==================

Ensure responses adhere to a JSON schema.

Try it out
----------

Try it out in the [Playground](/playground) or generate a ready-to-use schema definition to experiment with structured outputs.

Generate

Introduction
------------

JSON is one of the most widely used formats in the world for applications to exchange data.

Structured Outputs is a feature that ensures the model will always generate responses that adhere to your supplied [JSON Schema](https://json-schema.org/overview/what-is-jsonschema), so you don't need to worry about the model omitting a required key, or hallucinating an invalid enum value.

Some benefits of Structured Outputs include:

1.  **Reliable type-safety:** No need to validate or retry incorrectly formatted responses
2.  **Explicit refusals:** Safety-based model refusals are now programmatically detectable
3.  **Simpler prompting:** No need for strongly worded prompts to achieve consistent formatting

Getting a structured response

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const response = await openai.responses.create({
  model: "gpt-4o-2024-08-06",
  input: [
    {"role": "system", "content": "Extract the event information."},
    {"role": "user", "content": "Alice and Bob are going to a science fair on Friday."}
  ],
  text: {
    format: {
      type: "json_schema",
      name: "calendar_event",
      schema: {
        type: "object",
        properties: {
          name: { 
            type: "string" 
          },
          date: { 
            type: "string" 
          },
          participants: { 
            type: "array", 
            items: { 
              type: "string" 
            } 
          },
        },
        required: ["name", "date", "participants"],
        additionalProperties: false,
      },
    }
  }
});

const event = JSON.parse(response.output_text);
```

```python
from openai import OpenAI
import json

client = OpenAI()

response = client.responses.create(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "Extract the event information."},
        {"role": "user", "content": "Alice and Bob are going to a science fair on Friday."}
    ],
    text={
        "format": {
            "type": "json_schema",
            "name": "calendar_event",
            "schema": {
                "type": "object",
                "properties": {
                    "name": {
                        "type": "string"
                    },
                    "date": {
                        "type": "string"
                    },
                    "participants": {
                        "type": "array", 
                        "items": {
                            "type": "string"
                        }
                    },
                },
                "required": ["name", "date", "participants"],
                "additionalProperties": False
            },
            "strict": True
        }
    }
)

event = json.loads(response.output_text)
```

### Supported models

Structured Outputs is available in our [latest large language models](/docs/models), starting with GPT-4o:

*   `gpt-4.5-preview-2025-02-27` and later
*   `o3-mini-2025-1-31` and later
*   `o1-2024-12-17` and later
*   `gpt-4o-mini-2024-07-18` and later
*   `gpt-4o-2024-08-06` and later

Older models like `gpt-4-turbo` and earlier may use [JSON mode](#json-mode) instead.

When to use Structured Outputs via function calling vs via text.format
----------------------------------------------------------------------

Structured Outputs is available in two forms in the OpenAI API:

1.  When using [function calling](/docs/guides/function-calling)
2.  When using a `json_schema` response format

Function calling is useful when you are building an application that bridges the models and functionality of your application.

For example, you can give the model access to functions that query a database in order to build an AI assistant that can help users with their orders, or functions that can interact with the UI.

Conversely, Structured Outputs via `response_format` are more suitable when you want to indicate a structured schema for use when the model responds to the user, rather than when the model calls a tool.

For example, if you are building a math tutoring application, you might want the assistant to respond to your user using a specific JSON Schema so that you can generate a UI that displays different parts of the model's output in distinct ways.

Put simply:

*   If you are connecting the model to tools, functions, data, etc. in your system, then you should use function calling
*   If you want to structure the model's output when it responds to the user, then you should use a structured `text.format`

The remainder of this guide will focus on non-function calling use cases in the Responses API. To learn more about how to use Structured Outputs with function calling, check out the [Function Calling](/docs/guides/function-calling#function-calling-with-structured-outputs) guide.

### Structured Outputs vs JSON mode

Structured Outputs is the evolution of [JSON mode](#json-mode). While both ensure valid JSON is produced, only Structured Outputs ensure schema adherance. Both Structured Outputs and JSON mode are supported in the Responses API,Chat Completions API, Assistants API, Fine-tuning API and Batch API.

We recommend always using Structured Outputs instead of JSON mode when possible.

However, Structured Outputs with `response_format: {type: "json_schema", ...}` is only supported with the `gpt-4o-mini`, `gpt-4o-mini-2024-07-18`, and `gpt-4o-2024-08-06` model snapshots and later.

||Structured Outputs|JSON Mode|
|---|---|---|
|Outputs valid JSON|Yes|Yes|
|Adheres to schema|Yes (see supported schemas)|No|
|Compatible models|gpt-4o-mini, gpt-4o-2024-08-06, and later|gpt-3.5-turbo, gpt-4-* and gpt-4o-* models|
|Enabling|text: { format: { type: "json_schema", "strict": true, "schema": ... } }|text: { format: { type: "json_object" } }|

Examples
--------

Chain of thought

### Chain of thought

You can ask the model to output an answer in a structured, step-by-step way, to guide the user through the solution.

Structured Outputs for chain-of-thought math tutoring

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const response = await openai.responses.create({
    model: "gpt-4o-2024-08-06",
    input: [
        {
            role: "system",
            content:
                "You are a helpful math tutor. Guide the user through the solution step by step.",
        },
        { role: "user", content: "how can I solve 8x + 7 = -23" },
    ],
    text: {
        format: {
            type: "json_schema",
            name: "math_reasoning",
            schema: {
                type: "object",
                properties: {
                    steps: {
                        type: "array",
                        items: {
                            type: "object",
                            properties: {
                                explanation: { type: "string" },
                                output: { type: "string" },
                            },
                            required: ["explanation", "output"],
                            additionalProperties: false,
                        },
                    },
                    final_answer: { type: "string" },
                },
                required: ["steps", "final_answer"],
                additionalProperties: false,
            },
            strict: true,
        },
    },
});

const math_reasoning = JSON.parse(response.output_text);
```

```python
import json
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
        {"role": "user", "content": "how can I solve 8x + 7 = -23"}
    ],
    text={
        "format": {
            "type": "json_schema",
            "name": "math_reasoning",
            "schema": {
                "type": "object",
                "properties": {
                    "steps": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "explanation": { "type": "string" },
                                "output": { "type": "string" }
                            },
                            "required": ["explanation", "output"],
                            "additionalProperties": False
                        }
                    },
                    "final_answer": { "type": "string" }
                },
                "required": ["steps", "final_answer"],
                "additionalProperties": False
            },
            "strict": True
        }
    }
)

math_reasoning = json.loads(response.output_text)
```

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-2024-08-06",
    "input": [
      {
        "role": "system",
        "content": "You are a helpful math tutor. Guide the user through the solution step by step."
      },
      {
        "role": "user",
        "content": "how can I solve 8x + 7 = -23"
      }
    ],
    "text": {
      "format": {
        "type": "json_schema",
        "name": "math_reasoning",
        "schema": {
          "type": "object",
          "properties": {
            "steps": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "explanation": { "type": "string" },
                  "output": { "type": "string" }
                },
                "required": ["explanation", "output"],
                "additionalProperties": false
              }
            },
            "final_answer": { "type": "string" }
          },
          "required": ["steps", "final_answer"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

#### Example response

```json
{
  "steps": [
    {
      "explanation": "Start with the equation 8x + 7 = -23.",
      "output": "8x + 7 = -23"
    },
    {
      "explanation": "Subtract 7 from both sides to isolate the term with the variable.",
      "output": "8x = -23 - 7"
    },
    {
      "explanation": "Simplify the right side of the equation.",
      "output": "8x = -30"
    },
    {
      "explanation": "Divide both sides by 8 to solve for x.",
      "output": "x = -30 / 8"
    },
    {
      "explanation": "Simplify the fraction.",
      "output": "x = -15 / 4"
    }
  ],
  "final_answer": "x = -15 / 4"
}
```

Structured data extraction

### Structured data extraction

You can define structured fields to extract from unstructured input data, such as research papers.

Extracting data from research papers using Structured Outputs

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const response = await openai.responses.create({
    model: "gpt-4o-2024-08-06",
    input: [
        {
            role: "system",
            content:
                "You are an expert at structured data extraction. You will be given unstructured text from a research paper and should convert it into the given structure.",
        },
        { role: "user", content: "..." },
    ],
    text: {
        format: {
            type: "json_schema",
            name: "research_paper_extraction",
            schema: {
                type: "object",
                properties: {
                    title: { type: "string" },
                    authors: {
                        type: "array",
                        items: { type: "string" },
                    },
                    abstract: { type: "string" },
                    keywords: {
                        type: "array",
                        items: { type: "string" },
                    },
                },
                required: ["title", "authors", "abstract", "keywords"],
                additionalProperties: false,
            },
            strict: true,
        },
    },
});

const research_paper = JSON.parse(response.output_text);
```

```python
import json
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "You are an expert at structured data extraction. You will be given unstructured text from a research paper and should convert it into the given structure."},
        {"role": "user", "content": "..."}
    ],
    text={
        "format": {
              "type": "json_schema",
              "name": "research_paper_extraction",
              "schema": {
                "type": "object",
                "properties": {
                    "title": { "type": "string" },
                    "authors": { 
                        "type": "array",
                        "items": { "type": "string" }
                    },
                    "abstract": { "type": "string" },
                    "keywords": { 
                        "type": "array", 
                        "items": { "type": "string" }
                    }
                },
                "required": ["title", "authors", "abstract", "keywords"],
                "additionalProperties": False
            },
            "strict": True
        },
    },
)

research_paper = json.loads(response.output_text)
```

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-2024-08-06",
    "input": [
      {
        "role": "system",
        "content": "You are an expert at structured data extraction. You will be given unstructured text from a research paper and should convert it into the given structure."
      },
      {
        "role": "user",
        "content": "..."
      }
    ],
    "text": {
      "format": {
        "type": "json_schema",
        "name": "research_paper_extraction",
        "schema": {
          "type": "object",
          "properties": {
            "title": { "type": "string" },
            "authors": {
              "type": "array",
              "items": { "type": "string" }
            },
            "abstract": { "type": "string" },
            "keywords": {
              "type": "array",
              "items": { "type": "string" }
            }
          },
          "required": ["title", "authors", "abstract", "keywords"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

#### Example response

```json
{
  "title": "Application of Quantum Algorithms in Interstellar Navigation: A New Frontier",
  "authors": [
    "Dr. Stella Voyager",
    "Dr. Nova Star",
    "Dr. Lyra Hunter"
  ],
  "abstract": "This paper investigates the utilization of quantum algorithms to improve interstellar navigation systems. By leveraging quantum superposition and entanglement, our proposed navigation system can calculate optimal travel paths through space-time anomalies more efficiently than classical methods. Experimental simulations suggest a significant reduction in travel time and fuel consumption for interstellar missions.",
  "keywords": [
    "Quantum algorithms",
    "interstellar navigation",
    "space-time anomalies",
    "quantum superposition",
    "quantum entanglement",
    "space travel"
  ]
}
```

UI generation

### UI Generation

You can generate valid HTML by representing it as recursive data structures with constraints, like enums.

Generating HTML using Structured Outputs

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const response = await openai.responses.create({
    model: "gpt-4o-2024-08-06",
    input: [
        {
            role: "system",
            content:
                "You are a UI generator AI. Convert the user input into a UI.",
        },
        {
            role: "user",
            content: "Make a User Profile Form",
        },
    ],
    text: {
        format: {
            type: "json_schema",
            name: "ui",
            description: "Dynamically generated UI",
            schema: {
                type: "object",
                properties: {
                    type: {
                        type: "string",
                        description: "The type of the UI component",
                        enum: [
                            "div",
                            "button",
                            "header",
                            "section",
                            "field",
                            "form",
                        ],
                    },
                    label: {
                        type: "string",
                        description:
                            "The label of the UI component, used for buttons or form fields",
                    },
                    children: {
                        type: "array",
                        description: "Nested UI components",
                        items: { $ref: "#" },
                    },
                    attributes: {
                        type: "array",
                        description:
                            "Arbitrary attributes for the UI component, suitable for any element",
                        items: {
                            type: "object",
                            properties: {
                                name: {
                                    type: "string",
                                    description:
                                        "The name of the attribute, for example onClick or className",
                                },
                                value: {
                                    type: "string",
                                    description: "The value of the attribute",
                                },
                            },
                            required: ["name", "value"],
                            additionalProperties: false,
                        },
                    },
                },
                required: ["type", "label", "children", "attributes"],
                additionalProperties: false,
            },
            strict: true,
        },
    },
});

const ui = JSON.parse(response.output_text);
```

```python
import json
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "You are a UI generator AI. Convert the user input into a UI."},
        {"role": "user", "content": "Make a User Profile Form"}
    ],
    text={
        "format": {
            "type": "json_schema",
            "name": "ui",
            "description": "Dynamically generated UI",
            "schema": {
                "type": "object",
                "properties": {
                    "type": {
                        "type": "string",
                        "description": "The type of the UI component",
                        "enum": ["div", "button", "header", "section", "field", "form"]
                    },
                    "label": {
                        "type": "string",
                        "description": "The label of the UI component, used for buttons or form fields"
                    },
                    "children": {
                        "type": "array",
                        "description": "Nested UI components",
                        "items": {"$ref": "#"}
                    },
                    "attributes": {
                        "type": "array",
                        "description": "Arbitrary attributes for the UI component, suitable for any element",
                        "items": {
                            "type": "object",
                            "properties": {
                              "name": {
                                  "type": "string",
                                  "description": "The name of the attribute, for example onClick or className"
                              },
                              "value": {
                                  "type": "string",
                                  "description": "The value of the attribute"
                              }
                          },
                          "required": ["name", "value"],
                          "additionalProperties": False
                      }
                    }
                },
                "required": ["type", "label", "children", "attributes"],
                "additionalProperties": False
            },
            "strict": True,
        },
    },
)

ui = json.loads(response.output_text)
```

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-2024-08-06",
    "input": [
      {
        "role": "system",
        "content": "You are a UI generator AI. Convert the user input into a UI."
      },
      {
        "role": "user",
        "content": "Make a User Profile Form"
      }
    ],
    "text": {
      "format": {
        "type": "json_schema",
        "name": "ui",
        "description": "Dynamically generated UI",
        "schema": {
          "type": "object",
          "properties": {
            "type": {
              "type": "string",
              "description": "The type of the UI component",
              "enum": ["div", "button", "header", "section", "field", "form"]
            },
            "label": {
              "type": "string",
              "description": "The label of the UI component, used for buttons or form fields"
            },
            "children": {
              "type": "array",
              "description": "Nested UI components",
              "items": {"$ref": "#"}
            },
            "attributes": {
              "type": "array",
              "description": "Arbitrary attributes for the UI component, suitable for any element",
              "items": {
                "type": "object",
                "properties": {
                  "name": {
                    "type": "string",
                    "description": "The name of the attribute, for example onClick or className"
                  },
                  "value": {
                    "type": "string",
                    "description": "The value of the attribute"
                  }
                },
                "required": ["name", "value"],
                "additionalProperties": false
              }
            }
          },
          "required": ["type", "label", "children", "attributes"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

#### Example response

```json
{
    "type": "form",
    "label": "User Profile Form",
    "children": [
        {
            "type": "div",
            "label": "",
            "children": [
                {
                    "type": "field",
                    "label": "First Name",
                    "children": [],
                    "attributes": [
                        {
                            "name": "type",
                            "value": "text"
                        },
                        {
                            "name": "name",
                            "value": "firstName"
                        },
                        {
                            "name": "placeholder",
                            "value": "Enter your first name"
                        }
                    ]
                },
                {
                    "type": "field",
                    "label": "Last Name",
                    "children": [],
                    "attributes": [
                        {
                            "name": "type",
                            "value": "text"
                        },
                        {
                            "name": "name",
                            "value": "lastName"
                        },
                        {
                            "name": "placeholder",
                            "value": "Enter your last name"
                        }
                    ]
                }
            ],
            "attributes": []
        },
        {
            "type": "button",
            "label": "Submit",
            "children": [],
            "attributes": [
                {
                    "name": "type",
                    "value": "submit"
                }
            ]
        }
    ],
    "attributes": [
        {
            "name": "method",
            "value": "post"
        },
        {
            "name": "action",
            "value": "/submit-profile"
        }
    ]
}
```

Moderation

### Moderation

You can classify inputs on multiple categories, which is a common way of doing moderation.

Moderation using Structured Outputs

```javascript
import OpenAI from "openai";

const openai = new OpenAI();

const response = await openai.responses.create({
    model: "gpt-4o-2024-08-06",
    input: [
      {
        "role": "system",
        "content": "Determine if the user input violates specific guidelines and explain if they do."
      },
      {
        "role": "user",
        "content": "How do I prepare for a job interview?"
      }
    ],
    text: {
        format: {
          "type": "json_schema",
          "name": "content_compliance",
          "description": "Determines if content is violating specific moderation rules",
          "schema": {
            "type": "object",
            "properties": {
              "is_violating": {
                "type": "boolean",
                "description": "Indicates if the content is violating guidelines"
              },
              "category": {
                "type": ["string", "null"],
                "description": "Type of violation, if the content is violating guidelines. Null otherwise.",
                "enum": ["violence", "sexual", "self_harm"]
              },
              "explanation_if_violating": {
                "type": ["string", "null"],
                "description": "Explanation of why the content is violating"
              }
            },
            "required": ["is_violating", "category", "explanation_if_violating"],
            "additionalProperties": false
          },
          "strict": true
        },
    },
});

const compliance = JSON.parse(response.output_text);
```

```python
import json
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "Determine if the user input violates specific guidelines and explain if they do."},
        {"role": "user", "content": "How do I prepare for a job interview?"}
    ],
    text={
        "format": {
            "type": "json_schema",
            "name": "content_compliance",
            "description": "Determines if content is violating specific moderation rules",
            "schema": {
            "type": "object",
            "properties": {
                "is_violating": {
                    "type": "boolean",
                   "description": "Indicates if the content is violating guidelines"
                },
                "category": {
                "type": ["string", "null"],
                    "description": "Type of violation, if the content is violating guidelines. Null otherwise.",
                    "enum": ["violence", "sexual", "self_harm"]
                },
                "explanation_if_violating": {
                    "type": ["string", "null"],
                    "description": "Explanation of why the content is violating"
                }
            },
                "required": ["is_violating", "category", "explanation_if_violating"],
                "additionalProperties": False
            },
            "strict": True
        },
    },
)

compliance = json.loads(response.output_text)
```

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-2024-08-06",
    "input": [
      {
        "role": "system",
        "content": "Determine if the user input violates specific guidelines and explain if they do."
      },
      {
        "role": "user",
        "content": "How do I prepare for a job interview?"
      }
    ],
    "text": {
      "format": {
        "type": "json_schema",
        "name": "content_compliance",
        "description": "Determines if content is violating specific moderation rules",
        "schema": {
          "type": "object",
          "properties": {
            "is_violating": {
              "type": "boolean",
              "description": "Indicates if the content is violating guidelines"
            },
            "category": {
              "type": ["string", "null"],
              "description": "Type of violation, if the content is violating guidelines. Null otherwise.",
              "enum": ["violence", "sexual", "self_harm"]
            },
            "explanation_if_violating": {
              "type": ["string", "null"],
              "description": "Explanation of why the content is violating"
            }
          },
          "required": ["is_violating", "category", "explanation_if_violating"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

#### Example response

```json
{
  "is_violating": false,
  "category": null,
  "explanation_if_violating": null
}
```

How to use Structured Outputs with text.format
----------------------------------------------

Step 1: Define your schema

First you must design the JSON Schema that the model should be constrained to follow. See the [examples](/docs/guides/structured-outputs#examples) at the top of this guide for reference.

While Structured Outputs supports much of JSON Schema, some features are unavailable either for performance or technical reasons. See [here](/docs/guides/structured-outputs#supported-schemas) for more details.

#### Tips for your JSON Schema

To maximize the quality of model generations, we recommend the following:

*   Name keys clearly and intuitively
*   Create clear titles and descriptions for important keys in your structure
*   Create and use evals to determine the structure that works best for your use case

Step 2: Supply your schema in the API call

To use Structured Outputs, simply specify

```json
text: { format: { type: "json_schema", "strict": true, "schema": … } }
```

For example:

```python
response = client.responses.create(
    model="gpt-4o-2024-08-06",
    input=[
        {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
        {"role": "user", "content": "how can I solve 8x + 7 = -23"}
    ],
    text={
        "format": {
            "type": "json_schema",
            "name": "calendar_event",
            "schema": {
                "type": "object",
                "properties": {
                    "steps": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "explanation": {"type": "string"},
                                "output": {"type": "string"}
                            },
                            "required": ["explanation", "output"],
                            "additionalProperties": False
                        }
                    },
                    "final_answer": {"type": "string"}
                },
                "required": ["steps", "final_answer"],
                "additionalProperties": False
            },
            "strict": True
        }
    }
)

print(response.output_text)
```

```javascript
const response = await openai.responses.create({
    model: "gpt-4o-2024-08-06",
    input: [
        { role: "system", content: "You are a helpful math tutor. Guide the user through the solution step by step." },
        { role: "user", content: "how can I solve 8x + 7 = -23" }
    ],
    text: {
        format: {
            type: "json_schema",
            name: "math_response",
            schema: {
                type: "object",
                properties: {
                    steps: {
                        type: "array",
                        items: {
                            type: "object",
                            properties: {
                                explanation: { type: "string" },
                                output: { type: "string" }
                            },
                            required: ["explanation", "output"],
                            additionalProperties: false
                        }
                    },
                    final_answer: { type: "string" }
                },
                required: ["steps", "final_answer"],
                additionalProperties: false
            },
            strict: true
        }
    }
});

console.log(response.output_text);
```

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-2024-08-06",
    "input": [
      {
        "role": "system",
        "content": "You are a helpful math tutor. Guide the user through the solution step by step."
      },
      {
        "role": "user",
        "content": "how can I solve 8x + 7 = -23"
      }
    ],
    "text": {
      "format": {
        "type": "json_schema",
        "name": "math_response",
        "schema": {
          "type": "object",
          "properties": {
            "steps": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "explanation": { "type": "string" },
                  "output": { "type": "string" }
                },
                "required": ["explanation", "output"],
                "additionalProperties": false
              }
            },
            "final_answer": { "type": "string" }
          },
          "required": ["steps", "final_answer"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

**Note:** the first request you make with any schema will have additional latency as our API processes the schema, but subsequent requests with the same schema will not have additional latency.

Step 3: Handle edge cases

In some cases, the model might not generate a valid response that matches the provided JSON schema.

This can happen in the case of a refusal, if the model refuses to answer for safety reasons, or if for example you reach a max tokens limit and the response is incomplete.

```javascript
try {
  const response = await openai.responses.create({
    model: "gpt-4o-2024-08-06",
    input: [{
        role: "system",
        content: "You are a helpful math tutor. Guide the user through the solution step by step.",
      },
      {
        role: "user",
        content: "how can I solve 8x + 7 = -23"
      },
    ],
    max_output_tokens: 50,
    text: {
      format: {
        type: "json_schema",
        name: "math_response",
        schema: {
          type: "object",
          properties: {
            steps: {
              type: "array",
              items: {
                type: "object",
                properties: {
                  explanation: {
                    type: "string"
                  },
                  output: {
                    type: "string"
                  },
                },
                required: ["explanation", "output"],
                additionalProperties: false,
              },
            },
            final_answer: {
              type: "string"
            },
          },
          required: ["steps", "final_answer"],
          additionalProperties: false,
        },
        strict: true,
      },
    }
  });

  if (response.status === "incomplete" && response.incomplete_details.reason === "max_output_tokens") {
    // Handle the case where the model did not return a complete response
    throw new Error("Incomplete response");
  }

  const math_response = response.output[0].content[0];

  if (math_response.type === "refusal") {
    // handle refusal
    console.log(math_response.refusal);
  } else if (math_response.type === "output_text") {
    console.log(math_response.text);
  } else {
    throw new Error("No response content");
  }
} catch (e) {
  // Handle edge cases
  console.error(e);
}
```

```python
try:
    response = client.responses.create(
        model="gpt-4o-2024-08-06",
        input=[
            {
                "role": "system",
                "content": "You are a helpful math tutor. Guide the user through the solution step by step.",
            },
            {"role": "user", "content": "how can I solve 8x + 7 = -23"},
        ],
        text={
            "format": {
                "type": "json_schema",
                "name": "math_response",
                "strict": True,
                "schema": {
                    "type": "object",
                    "properties": {
                        "steps": {
                            "type": "array",
                            "items": {
                                "type": "object",
                                "properties": {
                                    "explanation": {"type": "string"},
                                    "output": {"type": "string"},
                                },
                                "required": ["explanation", "output"],
                                "additionalProperties": False,
                            },
                        },
                        "final_answer": {"type": "string"},
                    },
                    "required": ["steps", "final_answer"],
                    "additionalProperties": False,
                },
                "strict": True,
            },
        },
    )
except Exception as e:
    # handle errors like finish_reason, refusal, content_filter, etc.
    pass
```

### Refusals with Structured Outputs

When using Structured Outputs with user-generated input, OpenAI models may occasionally refuse to fulfill the request for safety reasons. Since a refusal does not necessarily follow the schema you have supplied in `response_format`, the API response will include a new field called `refusal` to indicate that the model refused to fulfill the request.

When the `refusal` property appears in your output object, you might present the refusal in your UI, or include conditional logic in code that consumes the response to handle the case of a refused request.

```python
class Step(BaseModel):
    explanation: str
    output: str

class MathReasoning(BaseModel):
    steps: list[Step]
    final_answer: str

completion = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "You are a helpful math tutor. Guide the user through the solution step by step."},
        {"role": "user", "content": "how can I solve 8x + 7 = -23"}
    ],
    response_format=MathReasoning,
)

math_reasoning = completion.choices[0].message

# If the model refuses to respond, you will get a refusal message
if (math_reasoning.refusal):
    print(math_reasoning.refusal)
else:
    print(math_reasoning.parsed)
```

```javascript
const Step = z.object({
  explanation: z.string(),
  output: z.string(),
});

const MathReasoning = z.object({
  steps: z.array(Step),
  final_answer: z.string(),
});

const completion = await openai.beta.chat.completions.parse({
  model: "gpt-4o-2024-08-06",
  messages: [
    { role: "system", content: "You are a helpful math tutor. Guide the user through the solution step by step." },
    { role: "user", content: "how can I solve 8x + 7 = -23" },
  ],
  response_format: zodResponseFormat(MathReasoning, "math_reasoning"),
});

const math_reasoning = completion.choices[0].message

// If the model refuses to respond, you will get a refusal message
if (math_reasoning.refusal) {
  console.log(math_reasoning.refusal);
} else {
  console.log(math_reasoning.parsed);
}
```

The API response from a refusal will look something like this:

```json
{
  "id": "resp_1234567890",
  "object": "response",
  "created_at": 1721596428,
  "status": "completed",
  "error": null,
  "incomplete_details": null,
  "input": [],
  "instructions": null,
  "max_output_tokens": null,
  "model": "gpt-4o-2024-08-06",
  "output": [{
    "id": "msg_1234567890",
    "type": "message",
    "role": "assistant",
    "content": [
      {
        "type": "refusal",
        "refusal": "I'm sorry, I cannot assist with that request."
      }
    ]
  }],
  "usage": {
    "input_tokens": 81,
    "output_tokens": 11,
    "total_tokens": 92,
    "output_tokens_details": {
      "reasoning_tokens": 0,
    }
  },
}
```

### Tips and best practices

#### Handling user-generated input

If your application is using user-generated input, make sure your prompt includes instructions on how to handle situations where the input cannot result in a valid response.

The model will always try to adhere to the provided schema, which can result in hallucinations if the input is completely unrelated to the schema.

You could include language in your prompt to specify that you want to return empty parameters, or a specific sentence, if the model detects that the input is incompatible with the task.

#### Handling mistakes

Structured Outputs can still contain mistakes. If you see mistakes, try adjusting your instructions, providing examples in the system instructions, or splitting tasks into simpler subtasks. Refer to the [prompt engineering guide](/docs/guides/prompt-engineering) for more guidance on how to tweak your inputs.

Streaming
---------

You can use streaming to process model responses or function call arguments as they are being generated, and parse them as structured data.

That way, you don't have to wait for the entire response to complete before handling it. This is particularly useful if you would like to display JSON fields one by one, or handle function call arguments as soon as they are available.

We recommend relying on the SDKs to handle streaming with Structured Outputs.

```python
from openai import OpenAI

client = OpenAI()

stream = client.responses.create(
    model="gpt-4o",
    input=[
        {"role": "system", "content": "Extract entities from the input text"},
        {
            "role": "user",
            "content": "The quick brown fox jumps over the lazy dog with piercing blue eyes"
        },
    ],
    text={
        "format": {
            "type": "json_schema",
            "name": "entities",
            "schema": {
                "type": "object",
                "properties": {
                    "attributes": {
                        "type": "array",
                        "items": {"type": "string"}
                    },
                    "colors": {
                        "type": "array",
                        "items": {"type": "string"}
                    },
                    "animals": {
                        "type": "array",
                        "items": {"type": "string"}
                    }
                },
                "required": ["attributes", "colors", "animals"],
                "additionalProperties": False
            },
        }
    },
    stream=True,
)

for event in stream:
    if event.type == 'response.refusal.delta':
        print(event.delta, end="")
    elif event.type == 'response.output_text.delta':
        print(event.delta, end="")
    elif event.type == 'response.error':
        print(event.error, end="")
    elif event.type == 'response.completed':
        print("Completed")
        # print(event.response.output)
```

```javascript
import { OpenAI } from "openai";

const openai = new OpenAI();

const stream = await openai.responses.create({
    model: "gpt-4o",
    input: [{ role: "user", content: "What's the weather like in Paris today?" }],
    stream: true,
    text: {
        "format": {
            "type": "json_schema",
            "name": "entities",
            "schema": {
                "type": "object",
                "properties": {
                    "attributes": {
                        "type": "array",
                        "items": {"type": "string"}
                    },
                    "colors": {
                        "type": "array",
                        "items": {"type": "string"}
                    },
                    "animals": {
                        "type": "array",
                        "items": {"type": "string"}
                    }
                },
                "required": ["attributes", "colors", "animals"],
                "additionalProperties": false
            },
        }
    }
});

for await (const event of stream) {
    if (event.type === 'response.refusal.delta') {
        process.stdout.write(event.delta);
    } else if (event.type === 'response.output_text.delta') {
        process.stdout.write(event.delta);
    } else if (event.type === 'response.error') {
        process.stdout.write(event.error);  
    } else if (event.type === 'response.completed') {
        console.log("Completed")
        // console.log(event.response.output);
    }
}
```

Supported schemas
-----------------

Structured Outputs supports a subset of the [JSON Schema](https://json-schema.org/docs) language.

#### Supported types

The following types are supported for Structured Outputs:

*   String
*   Number
*   Boolean
*   Integer
*   Object
*   Array
*   Enum
*   anyOf

#### Root objects must not be `anyOf`

Note that the root level object of a schema must be an object, and not use `anyOf`. A pattern that appears in Zod (as one example) is using a discriminated union, which produces an `anyOf` at the top level. So code such as the following won't work:

```javascript
import { z } from 'zod';
import { zodResponseFormat } from 'openai/helpers/zod';

const BaseResponseSchema = z.object({ /* ... */ });
const UnsuccessfulResponseSchema = z.object({ /* ... */ });

const finalSchema = z.discriminatedUnion('status', [
    BaseResponseSchema,
    UnsuccessfulResponseSchema,
]);

// Invalid JSON Schema for Structured Outputs
const json = zodResponseFormat(finalSchema, 'final_schema');
```

#### All fields must be `required`

To use Structured Outputs, all fields or function parameters must be specified as `required`.

```json
{
    "name": "get_weather",
    "description": "Fetches the weather in the given location",
    "strict": true,
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The location to get the weather for"
            },
            "unit": {
                "type": "string",
                "description": "The unit to return the temperature in",
                "enum": ["F", "C"]
            }
        },
        "additionalProperties": false,
        "required": ["location", "unit"]
    }
}
```

Although all fields must be required (and the model will return a value for each parameter), it is possible to emulate an optional parameter by using a union type with `null`.

```json
{
    "name": "get_weather",
    "description": "Fetches the weather in the given location",
    "strict": true,
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The location to get the weather for"
            },
            "unit": {
                "type": ["string", "null"],
                "description": "The unit to return the temperature in",
                "enum": ["F", "C"]
            }
        },
        "additionalProperties": false,
        "required": [
            "location", "unit"
        ]
    }
}
```

#### Objects have limitations on nesting depth and size

A schema may have up to 100 object properties total, with up to 5 levels of nesting.

#### Limitations on total string size

In a schema, total string length of all property names, definition names, enum values, and const values cannot exceed 15,000 characters.

#### Limitations on enum size

A schema may have up to 500 enum values across all enum properties.

For a single enum property with string values, the total string length of all enum values cannot exceed 7,500 characters when there are more than 250 enum values.

#### `additionalProperties: false` must always be set in objects

`additionalProperties` controls whether it is allowable for an object to contain additional keys / values that were not defined in the JSON Schema.

Structured Outputs only supports generating specified keys / values, so we require developers to set `additionalProperties: false` to opt into Structured Outputs.

```json
{
    "name": "get_weather",
    "description": "Fetches the weather in the given location",
    "strict": true,
    "schema": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The location to get the weather for"
            },
            "unit": {
                "type": "string",
                "description": "The unit to return the temperature in",
                "enum": ["F", "C"]
            }
        },
        "additionalProperties": false,
        "required": [
            "location", "unit"
        ]
    }
}
```

#### Key ordering

When using Structured Outputs, outputs will be produced in the same order as the ordering of keys in the schema.

#### Some type-specific keywords are not yet supported

Notable keywords not supported include:

*   **For strings:** `minLength`, `maxLength`, `pattern`, `format`
*   **For numbers:** `minimum`, `maximum`, `multipleOf`
*   **For objects:** `patternProperties`, `unevaluatedProperties`, `propertyNames`, `minProperties`, `maxProperties`
*   **For arrays:** `unevaluatedItems`, `contains`, `minContains`, `maxContains`, `minItems`, `maxItems`, `uniqueItems`

If you turn on Structured Outputs by supplying `strict: true` and call the API with an unsupported JSON Schema, you will receive an error.

#### For `anyOf`, the nested schemas must each be a valid JSON Schema per this subset

Here's an example supported anyOf schema:

```json
{
    "type": "object",
    "properties": {
        "item": {
            "anyOf": [
                {
                    "type": "object",
                    "description": "The user object to insert into the database",
                    "properties": {
                        "name": {
                            "type": "string",
                            "description": "The name of the user"
                        },
                        "age": {
                            "type": "number",
                            "description": "The age of the user"
                        }
                    },
                    "additionalProperties": false,
                    "required": [
                        "name",
                        "age"
                    ]
                },
                {
                    "type": "object",
                    "description": "The address object to insert into the database",
                    "properties": {
                        "number": {
                            "type": "string",
                            "description": "The number of the address. Eg. for 123 main st, this would be 123"
                        },
                        "street": {
                            "type": "string",
                            "description": "The street name. Eg. for 123 main st, this would be main st"
                        },
                        "city": {
                            "type": "string",
                            "description": "The city of the address"
                        }
                    },
                    "additionalProperties": false,
                    "required": [
                        "number",
                        "street",
                        "city"
                    ]
                }
            ]
        }
    },
    "additionalProperties": false,
    "required": [
        "item"
    ]
}
```

#### Definitions are supported

You can use definitions to define subschemas which are referenced throughout your schema. The following is a simple example.

```json
{
    "type": "object",
    "properties": {
        "steps": {
            "type": "array",
            "items": {
                "$ref": "#/$defs/step"
            }
        },
        "final_answer": {
            "type": "string"
        }
    },
    "$defs": {
        "step": {
            "type": "object",
            "properties": {
                "explanation": {
                    "type": "string"
                },
                "output": {
                    "type": "string"
                }
            },
            "required": [
                "explanation",
                "output"
            ],
            "additionalProperties": false
        }
    },
    "required": [
        "steps",
        "final_answer"
    ],
    "additionalProperties": false
}
```

#### Recursive schemas are supported

Sample recursive schema using `#` to indicate root recursion.

```json
{
    "name": "ui",
    "description": "Dynamically generated UI",
    "strict": true,
    "schema": {
        "type": "object",
        "properties": {
            "type": {
                "type": "string",
                "description": "The type of the UI component",
                "enum": ["div", "button", "header", "section", "field", "form"]
            },
            "label": {
                "type": "string",
                "description": "The label of the UI component, used for buttons or form fields"
            },
            "children": {
                "type": "array",
                "description": "Nested UI components",
                "items": {
                    "$ref": "#"
                }
            },
            "attributes": {
                "type": "array",
                "description": "Arbitrary attributes for the UI component, suitable for any element",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {
                            "type": "string",
                            "description": "The name of the attribute, for example onClick or className"
                        },
                        "value": {
                            "type": "string",
                            "description": "The value of the attribute"
                        }
                    },
                    "additionalProperties": false,
                    "required": ["name", "value"]
                }
            }
        },
        "required": ["type", "label", "children", "attributes"],
        "additionalProperties": false
    }
}
```

Sample recursive schema using explicit recursion:

```json
{
    "type": "object",
    "properties": {
        "linked_list": {
            "$ref": "#/$defs/linked_list_node"
        }
    },
    "$defs": {
        "linked_list_node": {
            "type": "object",
            "properties": {
                "value": {
                    "type": "number"
                },
                "next": {
                    "anyOf": [
                        {
                            "$ref": "#/$defs/linked_list_node"
                        },
                        {
                            "type": "null"
                        }
                    ]
                }
            },
            "additionalProperties": false,
            "required": [
                "next",
                "value"
            ]
        }
    },
    "additionalProperties": false,
    "required": [
        "linked_list"
    ]
}
```

JSON mode
---------

JSON mode is a more basic version of the Structured Outputs feature. While JSON mode ensures that model output is valid JSON, Structured Outputs reliably matches the model's output to the schema you specify. We recommend you use Structured Outputs if it is supported for your use case.

When JSON mode is turned on, the model's output is ensured to be valid JSON, except for in some edge cases that you should detect and handle appropriately.

To turn on JSON mode with the Responses API you can set the `text.format` to `{ "type": "json_object" }`. If you are using function calling, JSON mode is always turned on.

Important notes:

*   When using JSON mode, you must always instruct the model to produce JSON via some message in the conversation, for example via your system message. If you don't include an explicit instruction to generate JSON, the model may generate an unending stream of whitespace and the request may run continually until it reaches the token limit. To help ensure you don't forget, the API will throw an error if the string "JSON" does not appear somewhere in the context.
*   JSON mode will not guarantee the output matches any specific schema, only that it is valid and parses without errors. You should use Structured Outputs to ensure it matches your schema, or if that is not possible, you should use a validation library and potentially retries to ensure that the output matches your desired schema.
*   Your application must detect and handle the edge cases that can result in the model output not being a complete JSON object (see below)

Handling edge cases

```javascript
const we_did_not_specify_stop_tokens = true;

try {
  const response = await openai.responses.create({
    model: "gpt-3.5-turbo-0125",
    input: [
      {
        role: "system",
        content: "You are a helpful assistant designed to output JSON.",
      },
      { role: "user", content: "Who won the world series in 2020? Please respond in the format {winner: ...}" },
    ],
    text: { format: { type: "json_object" } },
  });

  // Check if the conversation was too long for the context window, resulting in incomplete JSON 
  if (response.status === "incomplete" && response.incomplete_details.reason === "max_output_tokens") {
    // your code should handle this error case
  }

  // Check if the OpenAI safety system refused the request and generated a refusal instead
  if (response.output[0].content[0].type === "refusal") {
    // your code should handle this error case
    // In this case, the .content field will contain the explanation (if any) that the model generated for why it is refusing
    console.log(response.output[0].content[0].refusal)
  }

  // Check if the model's output included restricted content, so the generation of JSON was halted and may be partial
  if (response.status === "incomplete" && response.incomplete_details.reason === "content_filter") {
    // your code should handle this error case
  }

  if (response.status === "completed") {
    // In this case the model has either successfully finished generating the JSON object according to your schema, or the model generated one of the tokens you provided as a "stop token"

    if (we_did_not_specify_stop_tokens) {
      // If you didn't specify any stop tokens, then the generation is complete and the content key will contain the serialized JSON object
      // This will parse successfully and should now contain  {"winner": "Los Angeles Dodgers"}
      console.log(JSON.parse(response.output_text))
    } else {
      // Check if the response.output_text ends with one of your stop tokens and handle appropriately
    }
  }
} catch (e) {
  // Your code should handle errors here, for example a network error calling the API
  console.error(e)
}
```

```python
we_did_not_specify_stop_tokens = True

try:
    response = client.responses.create(
        model="gpt-3.5-turbo-0125",
        input=[
            {"role": "system", "content": "You are a helpful assistant designed to output JSON."},
            {"role": "user", "content": "Who won the world series in 2020? Please respond in the format {winner: ...}"}
        ],
        text={"format": {"type": "json_object"}}
    )

    # Check if the conversation was too long for the context window, resulting in incomplete JSON 
    if response.status == "incomplete" and response.incomplete_details.reason == "max_output_tokens":
        # your code should handle this error case
        pass

    # Check if the OpenAI safety system refused the request and generated a refusal instead
    if response.output[0].content[0].type == "refusal":
        # your code should handle this error case
        # In this case, the .content field will contain the explanation (if any) that the model generated for why it is refusing
        print(response.output[0].content[0]["refusal"])

    # Check if the model's output included restricted content, so the generation of JSON was halted and may be partial
    if response.status == "incomplete" and response.incomplete_details.reason == "content_filter":
        # your code should handle this error case
        pass

    if response.status == "completed":
        # In this case the model has either successfully finished generating the JSON object according to your schema, or the model generated one of the tokens you provided as a "stop token"

        if we_did_not_specify_stop_tokens:
            # If you didn't specify any stop tokens, then the generation is complete and the content key will contain the serialized JSON object
            # This will parse successfully and should now contain  "{"winner": "Los Angeles Dodgers"}"
            print(response.output_text)
        else:
            # Check if the response.output_text ends with one of your stop tokens and handle appropriately
            pass
except Exception as e:
    # Your code should handle errors here, for example a network error calling the API
    print(e)
```

Resources
---------

To learn more about Structured Outputs, we recommend browsing the following resources:

*   Check out our [introductory cookbook](https://cookbook.openai.com/examples/structured_outputs_intro) on Structured Outputs
*   Learn [how to build multi-agent systems](https://cookbook.openai.com/examples/structured_outputs_multi_agent) with Structured Outputs

Was this page useful?