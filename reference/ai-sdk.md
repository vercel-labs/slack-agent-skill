# AI SDK Reference

Comprehensive reference for using the Vercel AI SDK in Slack agent projects.

## Quick Start

### Installation

The template includes the core AI SDK packages. If starting fresh:

```bash
pnpm add ai @ai-sdk/gateway
```

### Basic Usage

```typescript
import { generateText } from "ai";
import { gateway } from "@ai-sdk/gateway";

const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,
  prompt: "Your prompt here",
});

console.log(result.text);
```

## Provider Options

### Option 1: Vercel AI Gateway (Recommended)

The easiest approach - no API keys needed when deployed on Vercel.

```typescript
import { gateway } from "@ai-sdk/gateway";

// OpenAI models
gateway("openai/gpt-4o-mini")
gateway("openai/gpt-4o")

// Anthropic models
gateway("anthropic/claude-sonnet-4-20250514")
gateway("anthropic/claude-3-5-haiku-latest")

// Google models
gateway("google/gemini-2.0-flash")
```

**Benefits:**
- Zero API key configuration
- Access to multiple providers through a single interface
- Built-in rate limiting and observability
- Automatic failover capabilities

### Option 2: Direct Provider SDKs

For more control or when not using Vercel.

**OpenAI:**
```bash
pnpm add @ai-sdk/openai
```
```typescript
import { openai } from "@ai-sdk/openai";
// Requires OPENAI_API_KEY env var
const model = openai("gpt-4o-mini");
```

**Anthropic:**
```bash
pnpm add @ai-sdk/anthropic
```
```typescript
import { anthropic } from "@ai-sdk/anthropic";
// Requires ANTHROPIC_API_KEY env var
const model = anthropic("claude-sonnet-4-20250514");
```

**Google:**
```bash
pnpm add @ai-sdk/google
```
```typescript
import { google } from "@ai-sdk/google";
// Requires GOOGLE_GENERATIVE_AI_API_KEY env var
const model = google("gemini-2.0-flash");
```

## Core Functions

### generateText

For single-turn text generation:

```typescript
import { generateText } from "ai";
import { gateway } from "@ai-sdk/gateway";

const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,
  prompt: "Explain quantum computing in simple terms",
});

console.log(result.text);
console.log(result.usage.inputTokens);
console.log(result.usage.outputTokens);
```

### streamText

For streaming responses (great for real-time Slack updates):

```typescript
import { streamText } from "ai";
import { gateway } from "@ai-sdk/gateway";

const result = await streamText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,
  prompt: "Write a short story",
});

for await (const chunk of result.textStream) {
  // Update Slack message progressively
  process.stdout.write(chunk);
}
```

### generateObject

For structured JSON output:

```typescript
import { generateObject } from "ai";
import { gateway } from "@ai-sdk/gateway";
import { z } from "zod";

const result = await generateObject({
  model: gateway("openai/gpt-4o-mini"),
  schema: z.object({
    name: z.string(),
    age: z.number(),
    hobbies: z.array(z.string()),
  }),
  prompt: "Generate a fictional person profile",
});

console.log(result.object); // Typed as { name: string, age: number, hobbies: string[] }
```

### Structured Output via generateText (v6)

In AI SDK v6, you can also generate structured output using `generateText` with the `output` option:

```typescript
import { generateText, Output } from "ai";
import { gateway } from "@ai-sdk/gateway";
import { z } from "zod";

const PersonSchema = z.object({
  name: z.string(),
  age: z.number(),
  hobbies: z.array(z.string()),
});

const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  output: Output.object({ schema: PersonSchema }),
  prompt: "Generate a fictional person profile",
});

console.log(result.object); // Typed output
```

Available `Output` types:
- `Output.object({ schema })` - Generate a single object
- `Output.array({ schema })` - Generate an array of objects
- `Output.choice({ options })` - Select from predefined options
- `Output.json()` - Generate raw JSON

## Tool Calling

Define tools that the AI can call:

```typescript
import { generateText, tool } from "ai";
import { gateway } from "@ai-sdk/gateway";
import { z } from "zod";

const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,
  tools: {
    getWeather: tool({
      description: "Get current weather for a location",
      inputSchema: z.object({
        location: z.string().describe("City name"),
        unit: z.enum(["celsius", "fahrenheit"]).optional(),
      }),
      execute: async ({ location, unit = "celsius" }) => {
        // Implement weather API call
        return { temperature: 22, unit, condition: "sunny" };
      },
    }),
    searchWeb: tool({
      description: "Search the web for information",
      inputSchema: z.object({
        query: z.string().describe("Search query"),
      }),
      execute: async ({ query }) => {
        // Implement search
        return { results: ["Result 1", "Result 2"] };
      },
    }),
  },
  prompt: "What's the weather in Seattle and find me some restaurants there?",
});
```

## Agent Patterns

### ToolLoopAgent Pattern

For complex multi-step interactions:

```typescript
import { generateText, tool, ToolLoopAgent, stepCountIs } from "ai";
import { gateway } from "@ai-sdk/gateway";
import { z } from "zod";

const agent = new ToolLoopAgent({
  model: gateway("openai/gpt-4o"),
  tools: {
    // Define your tools
  },
  system: "You are a helpful assistant.",
  stopWhen: stepCountIs(10), // v6: was maxIterations
});

const result = await agent.run("Complete this complex task");
```

### Type-Safe Agents with InferAgentUIMessage

For end-to-end type safety when consuming agents with `useChat`:

```typescript
import { ToolLoopAgent, InferAgentUIMessage } from "ai";

const agent = new ToolLoopAgent({
  model: gateway("openai/gpt-4o"),
  tools: { /* your tools */ },
  system: "You are a helpful assistant.",
});

// Infer message type for UI components
type AgentMessage = InferAgentUIMessage<typeof agent>;
```

### Multi-Turn Conversations

For maintaining conversation history:

```typescript
import { generateText } from "ai";
import { gateway } from "@ai-sdk/gateway";

const conversationHistory: Array<{ role: "user" | "assistant"; content: string }> = [];

async function chat(userMessage: string) {
  conversationHistory.push({ role: "user", content: userMessage });

  const result = await generateText({
    model: gateway("openai/gpt-4o-mini"),
    maxOutputTokens: 1000,
    messages: conversationHistory,
  });

  conversationHistory.push({ role: "assistant", content: result.text });
  return result.text;
}
```

## AI SDK v6 API Changes

If migrating from v4/v5:

| Old (v4/v5) | New (v6) |
|-------------|----------|
| `maxTokens` | `maxOutputTokens` |
| `result.usage.promptTokens` | `result.usage.inputTokens` |
| `result.usage.completionTokens` | `result.usage.outputTokens` |
| `parameters` (in tools) | `inputSchema` |
| `maxSteps` / `maxIterations` | `stopWhen: stepCountIs(n)` |
| `part.args` (tool parts) | `part.input` |
| `part.result` (tool parts) | `part.output` |
| `addToolResult` (useChat) | `addToolOutput` |

## Slack Integration Patterns

### Streaming to Slack

Update Slack messages progressively for a better UX:

```typescript
import { streamText } from "ai";
import { gateway } from "@ai-sdk/gateway";

async function streamToSlack(client: WebClient, channel: string, threadTs: string) {
  // Post initial "thinking" message
  const msg = await client.chat.postMessage({
    channel,
    thread_ts: threadTs,
    text: "Thinking...",
  });

  const result = await streamText({
    model: gateway("openai/gpt-4o-mini"),
    maxOutputTokens: 1000,
    prompt: "Your prompt",
  });

  let fullText = "";
  let lastUpdate = Date.now();
  const UPDATE_INTERVAL = 500; // ms

  for await (const chunk of result.textStream) {
    fullText += chunk;

    // Throttle updates to avoid rate limits
    if (Date.now() - lastUpdate > UPDATE_INTERVAL) {
      await client.chat.update({
        channel,
        ts: msg.ts!,
        text: fullText + "...",
      });
      lastUpdate = Date.now();
    }
  }

  // Final update
  await client.chat.update({
    channel,
    ts: msg.ts!,
    text: fullText,
  });
}
```

### Tool Results in Slack

Format tool results nicely:

```typescript
import { generateText, tool } from "ai";
import { gateway } from "@ai-sdk/gateway";

const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  tools: {
    lookupUser: tool({
      description: "Look up a Slack user",
      inputSchema: z.object({
        userId: z.string(),
      }),
      execute: async ({ userId }) => {
        const user = await client.users.info({ user: userId });
        return {
          name: user.user?.real_name,
          email: user.user?.profile?.email,
        };
      },
    }),
  },
  prompt: "Who is <@U123456>?",
});
```

## DevTools (Development Only)

Capture all AI SDK calls to a local JSON file for debugging:

```bash
pnpm add @ai-sdk/devtools
```

```typescript
import { withDevTools } from "@ai-sdk/devtools";

const model = withDevTools(gateway("openai/gpt-4o-mini"));
```

View captured calls at `.devtools/generations.json` or run `npx @ai-sdk/devtools` for a web UI at http://localhost:4983.

## Best Practices

### 1. Verify APIs Against Source

**CRITICAL: Always verify AI SDK APIs against the installed package - never rely on memory.** The AI SDK evolves rapidly and training data may be outdated.

```bash
# Check bundled documentation
grep "generateText" node_modules/ai/docs/

# Check source code
grep "generateText" node_modules/ai/src/

# Check available functions
ls node_modules/ai/docs/
```

### 2. Handle Errors Gracefully

```typescript
try {
  const result = await generateText({
    model: gateway("openai/gpt-4o-mini"),
    prompt: userMessage,
  });
  return result.text;
} catch (error) {
  if (error.message.includes("rate_limit")) {
    return "I'm receiving too many requests. Please try again in a moment.";
  }
  console.error("AI error:", error);
  return "Sorry, I encountered an error processing your request.";
}
```

### 3. Set Reasonable Limits

```typescript
const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,      // Limit response length
  temperature: 0.7,           // Control randomness
  prompt: userMessage,
});
```

### 4. Use Type-Safe Outputs

```typescript
import { generateObject } from "ai";
import { z } from "zod";

const TaskSchema = z.object({
  title: z.string(),
  priority: z.enum(["low", "medium", "high"]),
  dueDate: z.string().optional(),
});

const result = await generateObject({
  model: gateway("openai/gpt-4o-mini"),
  schema: TaskSchema,
  prompt: "Create a task from: Review PR by end of day",
});

// result.object is fully typed
console.log(result.object.title);
console.log(result.object.priority);
```

## Model ID Discovery

**CRITICAL: Never use model IDs from memory or training data.** Model IDs change frequently. Always fetch current IDs before writing code.

Get current available models:

```bash
# OpenAI models
curl -s https://ai-gateway.vercel.sh/v1/models | jq -r '[.data[] | select(.id | startswith("openai/")) | .id] | reverse | .[]'

# Anthropic models
curl -s https://ai-gateway.vercel.sh/v1/models | jq -r '[.data[] | select(.id | startswith("anthropic/")) | .id] | reverse | .[]'

# Google models
curl -s https://ai-gateway.vercel.sh/v1/models | jq -r '[.data[] | select(.id | startswith("google/")) | .id] | reverse | .[]'
```

Use the model with the highest version number for the latest capabilities.

## References

- [AI SDK Documentation](https://ai-sdk.dev)
- [Vercel AI Gateway](https://vercel.com/docs/ai-gateway)
- [AI SDK GitHub](https://github.com/vercel/ai)
- [AI SDK Skills](https://github.com/vercel/ai/tree/main/skills)
