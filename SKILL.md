---
name: slack-agent-dev
description: Use when working on Slack agent/bot code, Slack Bolt applications, or projects using the slack-agent-template. Provides development patterns, testing requirements, and quality standards.
version: 2.0.0
user-invocable: false
---

# Vercel Slack Agent Development Skill

You are working on a Slack agent project built with the Vercel Slack Agent Template. Follow these mandatory practices for all code changes.

## Project Stack

This project uses:
- **Server**: Nitro (H3-based) with file-based routing
- **Slack SDK**: Bolt for JavaScript v4.x
- **AI**: AI SDK v6 with @ai-sdk/gateway
- **Workflows**: Workflow DevKit for durable execution
- **Linting**: Biome
- **Package Manager**: pnpm

### Dependencies

```json
{
  "dependencies": {
    "ai": "^6.0.0",
    "@ai-sdk/gateway": "latest",
    "@slack/bolt": "^4.x",
    "zod": "^3.x"
  }
}
```

**Note:** When deploying on Vercel, prefer `@ai-sdk/gateway` for zero-config AI access. Use direct provider SDKs (`@ai-sdk/openai`, `@ai-sdk/anthropic`, etc.) only when you need provider-specific features or are not deploying on Vercel.

---

## Quality Standards (MANDATORY)

These quality requirements MUST be followed for every code change. There are no exceptions.

### After EVERY File Modification

1. **Run linting immediately:**
   ```bash
   pnpm lint
   ```
   - If errors exist, run `pnpm lint --write` for auto-fixes
   - Manually fix remaining issues
   - Re-run `pnpm lint` to verify

2. **Check for corresponding test file:**
   - If you modified `foo.ts`, check if `foo.test.ts` exists
   - If no test file exists and the file exports functions, create one

### Before Completing ANY Task

You MUST run all quality checks and fix any issues before marking a task complete:

```bash
# 1. TypeScript compilation - must pass
pnpm typecheck

# 2. Linting - must pass with no errors
pnpm lint

# 3. Tests - all tests must pass
pnpm test
```

**Do NOT complete a task if any of these fail.** Fix the issues first.

### Unit Tests Required

**For ANY code change, you MUST write or update unit tests.**

- **Location**: Co-located `*.test.ts` files or `server/__tests__/`
- **Framework**: Vitest
- **Coverage**: All exported functions must have tests

Example test structure:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { myFunction } from './my-module';

describe('myFunction', () => {
  it('should handle normal input', () => {
    expect(myFunction('input')).toBe('expected');
  });

  it('should handle edge cases', () => {
    expect(myFunction('')).toBe('default');
  });
});
```

### E2E Tests for User-Facing Changes

If you modify:
- Slack message handlers
- Slash commands
- Interactive components (buttons, modals)
- Bot responses

You MUST add or update E2E tests that verify the full flow.

---

## Events Endpoint Pattern (CRITICAL)

The events endpoint must handle BOTH JSON events AND form-urlencoded slash commands.

### Complete Events Handler

```typescript
// server/api/slack/events.post.ts
import { defineEventHandler, getHeader, readRawBody } from "h3";
import { app } from "../../app";

export default defineEventHandler(async (event) => {
  const rawBody = (await readRawBody(event)) ?? "";
  const contentType = getHeader(event, "content-type") ?? "";
  const isFormData = contentType.includes("application/x-www-form-urlencoded");

  // CRITICAL: Slash commands use form-urlencoded, NOT JSON
  if (isFormData) {
    const params = new URLSearchParams(rawBody);
    const formData: Record<string, string> = {};
    for (const [key, value] of params) {
      formData[key] = value;
    }

    if (formData["command"]) {
      // For async commands using respond(), return empty string
      // Returning ANY JSON causes "invalid_command_response" error
      let ackResponse: unknown = "";

      await app.processEvent({
        body: { ...formData, type: "slash_command" },
        ack: async (response) => {
          // Only set if ack() was called WITH a response (sync pattern)
          if (response !== undefined) {
            ackResponse = response;
          }
        },
        respond: async (response) => {
          // Post to response_url for async responses
          await fetch(formData["response_url"], {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(response),
          });
        },
      });

      return ackResponse; // Empty string = 200 OK with no body
    }
  }

  // JSON events (mentions, messages, interactions, etc.)
  const body = JSON.parse(rawBody);

  // Handle URL verification challenge
  if (body.type === "url_verification") {
    return { challenge: body.challenge };
  }

  // Process Slack events
  await app.processEvent({
    body,
    ack: async (response) => response,
  });

  return { ok: true };
});
```

### Key Points

1. **Check Content-Type** - Slash commands are `application/x-www-form-urlencoded`
2. **Parse with URLSearchParams** - NOT `JSON.parse()` for form data
3. **Return empty for async** - When using `respond()` after `ack()`, return `""` not JSON
4. **Handle url_verification** - Return the challenge for Slack's URL verification

---

## Implementation Gotchas

### 1. Slash Commands Use Form-Encoded Data, Not JSON

Slack sends slash commands as `application/x-www-form-urlencoded`, not JSON. Always check the content-type header:

```typescript
const contentType = getHeader(event, "content-type") ?? "";
if (contentType.includes("application/x-www-form-urlencoded")) {
  const formData = new URLSearchParams(rawBody);
  // Parse as slash command
}
```

### 2. Async Slash Commands Must Return Empty 200

When a slash command uses `respond()` asynchronously after `ack()`, the initial HTTP response **must be an empty string**, NOT a JSON object. Returning `{ ok: true }` causes `invalid_command_response` errors.

```typescript
// WRONG - causes error
await ack();
await respond({ text: "..." });
return { ok: true };

// CORRECT - empty 200 response
await ack();
await respond({ text: "..." });
return "";
```

### 3. Private Channel Access

Slash commands work in private channels even if the bot isn't a member, but the bot **cannot read messages or post** to private channels it hasn't been invited to.

When creating features that will later post to a channel:

```typescript
// Validate channel access upfront
const channelInfo = await client.conversations.info({ channel: channelId });

if (channelInfo.channel?.is_private && !channelInfo.channel?.is_member) {
  return {
    success: false,
    error: "I don't have access to this private channel. Please add me with `/invite @BotName` first.",
  };
}
```

### 4. Graceful Degradation for Channel Context

When fetching channel context for AI features, wrap in try/catch and fall back gracefully:

```typescript
let channelContext = "";
try {
  const history = await client.conversations.history({
    channel: channelId,
    limit: 10,
  });
  channelContext = history.messages?.map(m => m.text).join("\n") ?? "";
} catch (error) {
  // Bot can't access channel - continue without context
  console.log("Could not fetch channel context:", error);
}
```

### 5. Vercel Cron Endpoint Authentication

Protect cron endpoints with a `CRON_SECRET` environment variable:

```typescript
// server/api/cron/my-job.get.ts
export default defineEventHandler(async (event) => {
  const authHeader = getHeader(event, "authorization");

  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    setResponseStatus(event, 401);
    return { error: "Unauthorized" };
  }

  // Run cron job logic...
  return { success: true };
});
```

### 6. vercel.json Cron Configuration

Configure cron jobs in `vercel.json`:

```json
{
  "crons": [
    {
      "path": "/api/cron/my-job",
      "schedule": "0 * * * *"
    }
  ]
}
```

Schedule format is standard cron syntax: `minute hour day month weekday`

Common schedules:
- `* * * * *` - Every minute
- `0 * * * *` - Every hour
- `0 0 * * *` - Daily at midnight
- `0 9 * * 1-5` - Weekdays at 9am

---

## AI Integration

You have two options for AI/LLM integration in your Slack agent.

### Option 1: Vercel AI Gateway (Recommended)

Use the modern `@ai-sdk/gateway` package - NO API keys needed on Vercel!

### Basic Usage

```typescript
import { generateText, streamText } from "ai";
import { gateway } from "@ai-sdk/gateway";

// Simple text generation
const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,  // v6: was maxTokens
  prompt: "Your prompt here",
});

console.log(result.text);
console.log(result.usage.inputTokens);   // v6: was promptTokens
console.log(result.usage.outputTokens);  // v6: was completionTokens
```

### Streaming Responses

```typescript
const result = await streamText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,
  prompt: userMessage,
});

for await (const chunk of result.textStream) {
  // Stream to Slack via chat.update
}
```

### With Tools

```typescript
import { tool } from "ai";
import { z } from "zod";

const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  maxOutputTokens: 1000,
  tools: {
    getWeather: tool({
      description: "Get weather for a location",
      parameters: z.object({
        location: z.string().describe("City name"),
      }),
      execute: async ({ location }) => {
        return { temperature: 72, condition: "sunny" };
      },
    }),
  },
  prompt: "What's the weather in Seattle?",
});
```

### AI SDK v6 API Changes

| v4/v5 | v6 |
|-------|-----|
| `maxTokens` | `maxOutputTokens` |
| `result.usage.promptTokens` | `result.usage.inputTokens` |
| `result.usage.completionTokens` | `result.usage.outputTokens` |

### Option 2: Direct Provider SDK

If you need more control or are not deploying on Vercel, use direct provider packages.

**OpenAI:**
```bash
pnpm add @ai-sdk/openai
```
```typescript
import { generateText } from "ai";
import { openai } from "@ai-sdk/openai";

// Requires OPENAI_API_KEY env var
const result = await generateText({
  model: openai("gpt-4o-mini"),
  maxOutputTokens: 1000,
  prompt: "Your prompt here",
});
```

**Anthropic:**
```bash
pnpm add @ai-sdk/anthropic
```
```typescript
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";

// Requires ANTHROPIC_API_KEY env var
const result = await generateText({
  model: anthropic("claude-sonnet-4-20250514"),
  maxOutputTokens: 1000,
  prompt: "Your prompt here",
});
```

**Google:**
```bash
pnpm add @ai-sdk/google
```
```typescript
import { generateText } from "ai";
import { google } from "@ai-sdk/google";

// Requires GOOGLE_GENERATIVE_AI_API_KEY env var
const result = await generateText({
  model: google("gemini-2.0-flash"),
  maxOutputTokens: 1000,
  prompt: "Your prompt here",
});
```

**When to use direct providers:**
- Not deploying on Vercel
- Need provider-specific features not available through gateway
- Already managing API keys for other purposes
- Need maximum control over provider configuration

For comprehensive AI SDK documentation, see `./reference/ai-sdk.md`.

---

## Stateful Patterns with Vercel Workflow

When building Slack agents that need data persistence across multiple interactions—like multi-turn conversations, collaborative workflows, or story generation—use Vercel Workflow instead of a database.

### When to Use Workflow

**Use Vercel Workflow when:**
- Conversational flows span multiple messages
- Need to accumulate state across user interactions
- Building collaborative or multi-step workflows
- Don't need long-term persistence (workflow scope)

**Use a database when:**
- Need data to persist beyond the workflow
- Require querying historical data
- Multiple workflows need to share state

### The "use workflow" Directive

Vercel Workflow functions can run for extended periods (not limited to serverless timeout) and maintain state as local variables:

```typescript
import { serve } from "@anthropic-ai/sdk/workflows";

export const { POST } = serve(async function myWorkflow(params: URLSearchParams) {
  "use workflow";

  // State as local variables - persists across the entire workflow!
  const messages: Message[] = [];
  let conversationComplete = false;

  // Your workflow logic here...
  while (!conversationComplete) {
    // Wait for events, process, update state
  }

  return { messages, result: "done" };
});
```

### Event Subscriptions with defineHook

Use `defineHook` to subscribe to incoming Slack events within your workflow:

```typescript
import { defineHook } from "@anthropic-ai/sdk/workflows";
import { z } from "zod";

// Define the schema for incoming events
const slackMessageSchema = z.object({
  text: z.string(),
  user: z.string(),
  ts: z.string(),
  channel: z.string(),
});

export const messageHook = defineHook({ schema: slackMessageSchema });
```

### Complete Example: Multi-Turn Conversation

```typescript
// app/api/conversation/route.ts
import { serve } from "@anthropic-ai/sdk/workflows";
import { defineHook } from "@anthropic-ai/sdk/workflows";
import { generateText } from "ai";
import { gateway } from "@ai-sdk/gateway";
import { z } from "zod";

const messageSchema = z.object({
  text: z.string(),
  user: z.string(),
  ts: z.string(),
  channel: z.string(),
});

export const userMessageHook = defineHook({ schema: messageSchema });

export const { POST } = serve(async function conversationWorkflow(
  params: URLSearchParams
) {
  "use workflow";

  const channelId = params.get("channel_id")!;
  const userId = params.get("user_id")!;

  // State persists as local variables
  const conversationHistory: Array<{ role: string; content: string }> = [];
  let turnCount = 0;
  const maxTurns = 10;

  // Create event stream for this channel
  const eventStream = userMessageHook.create({
    channel: channelId,
    user: userId,
  });

  // Process messages as they arrive
  for await (const event of eventStream) {
    turnCount++;

    // Add user message to history
    conversationHistory.push({
      role: "user",
      content: event.text,
    });

    // Generate AI response
    const result = await generateText({
      model: gateway("anthropic/claude-sonnet-4-20250514"),
      maxOutputTokens: 1000,
      messages: conversationHistory,
    });

    // Add assistant response to history
    conversationHistory.push({
      role: "assistant",
      content: result.text,
    });

    // Post response to Slack (via your Slack client)
    await postToSlack(channelId, result.text, event.ts);

    // Check for conversation end conditions
    if (turnCount >= maxTurns || event.text.toLowerCase().includes("goodbye")) {
      break;
    }
  }

  return {
    turns: turnCount,
    history: conversationHistory,
  };
});
```

### Triggering Workflows from Slack Events

Start a workflow when a user initiates a conversation:

```typescript
// server/listeners/events/app-mention.ts
export function registerAppMention(app: App) {
  app.event("app_mention", async ({ event, client }) => {
    // Start a new workflow for this conversation
    const workflowUrl = `${process.env.VERCEL_URL}/api/conversation`;

    await fetch(workflowUrl, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        channel_id: event.channel,
        user_id: event.user,
        thread_ts: event.thread_ts || event.ts,
      }),
    });

    await client.chat.postMessage({
      channel: event.channel,
      thread_ts: event.thread_ts || event.ts,
      text: "Starting our conversation! I'll remember everything we discuss.",
    });
  });
}
```

### Posting Events to Running Workflows

Forward Slack messages to the running workflow's hook:

```typescript
// server/listeners/messages/thread-message.ts
export function registerThreadMessage(app: App) {
  app.message(async ({ message, client }) => {
    if (!message.thread_ts || "bot_id" in message) return;

    // Post to the workflow's hook endpoint
    await userMessageHook.post({
      text: message.text,
      user: message.user,
      ts: message.ts,
      channel: message.channel,
    });
  });
}
```

### Key Benefits

1. **No database setup** - State lives in workflow memory
2. **Extended execution** - Not limited by serverless timeouts
3. **Natural programming model** - Use loops and local variables
4. **Automatic persistence** - Vercel handles state durability

### Reference

- [Vercel Workflow Documentation](https://vercel.com/docs/workflow)
- [Stateful Slack Bots Guide](https://vercel.com/kb/guide/stateful-slack-bots-with-vercel-workflow)
- [Example: Storytime Slackbot](https://github.com/vercel-labs/storytime-slackbot)

---

## Code Organization

Follow the template's established patterns.

### Tools (`server/lib/ai/tools.ts`)

```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const myTool = tool({
  description: 'Clear description of what this tool does',
  parameters: z.object({
    param: z.string().describe('What this parameter is for'),
  }),
  execute: async ({ param }) => {
    // Implementation
    return { success: true, data: result };
  },
});
```

### Listeners (`server/listeners/`)

Organize by event type:
- `actions/` - Button clicks, menu selections
- `assistant/` - Slack Assistant events
- `commands/` - Slash commands
- `events/` - App events (mentions, joins)
- `messages/` - Message handling
- `shortcuts/` - Global/message shortcuts
- `views/` - Modal submissions

### Workflows (`server/lib/ai/workflows/`)

Use `defineWorkflow` for multi-step operations:
```typescript
import { defineWorkflow } from '@vercel/workflow-devkit';

export const myWorkflow = defineWorkflow({
  id: 'my-workflow',
  // ...
});
```

---

## Environment Variables

Required variables (access via `process.env`):
- `SLACK_BOT_TOKEN` - Bot OAuth token
- `SLACK_SIGNING_SECRET` - Request signing

Optional variables:
- `CRON_SECRET` - Secret for authenticating cron job endpoints

**No AI API keys needed!** Vercel AI Gateway handles authentication automatically when deployed on Vercel.

**Never hardcode credentials. Never commit `.env` files.**

---

## Slack-Specific Patterns

### Block Kit UI

Use Block Kit for rich messages:
```typescript
import { Blocks, Elements, Bits } from 'slack-block-builder';

const message = Blocks([
  Blocks.Section({ text: 'Hello!' }),
  Blocks.Actions([
    Elements.Button({ text: 'Click me', actionId: 'btn_click' })
  ])
]);
```

### Message Formatting

Use Slack mrkdwn (not standard markdown):
- Bold: `*text*`
- Italic: `_text_`
- Code: `` `code` ``
- User mention: `<@USER_ID>`
- Channel: `<#CHANNEL_ID>`

### Error Handling

Return structured responses:
```typescript
return {
  success: false,
  error: 'User-friendly error message',
  details: technicalDetails // for logging
};
```

For detailed Slack patterns, see `./patterns/slack-patterns.md`.

---

## Git Commit Standards

Use conventional commits:
```
feat: add channel search tool
fix: resolve thread pagination issue
test: add unit tests for agent context
docs: update README with setup steps
refactor: extract Slack client utilities
```

**Never commit:**
- `.env` files
- API keys or tokens
- `node_modules/`

---

## Quick Commands

```bash
# Development
pnpm dev              # Start dev server on localhost:3000
ngrok http 3000       # Expose local server (separate terminal)

# Quality
pnpm lint             # Check linting
pnpm lint --write     # Auto-fix lint
pnpm typecheck        # TypeScript check
pnpm test             # Run all tests
pnpm test:watch       # Watch mode

# Build & Deploy
pnpm build            # Build for production
vercel                # Deploy to Vercel
```

---

## Reference Documentation

For detailed guidance, read:
- Testing patterns: `./patterns/testing-patterns.md`
- Slack patterns: `./patterns/slack-patterns.md`
- Environment setup: `./reference/env-vars.md`
- AI SDK: `./reference/ai-sdk.md`
- Slack setup: `./reference/slack-setup.md`
- Vercel deployment: `./reference/vercel-setup.md`

---

## Checklist Before Task Completion

Before marking ANY task as complete, verify:

- [ ] Code changes have corresponding tests
- [ ] `pnpm lint` passes with no errors
- [ ] `pnpm typecheck` passes with no errors
- [ ] `pnpm test` passes with no failures
- [ ] No hardcoded credentials
- [ ] Follows existing code patterns
- [ ] Events endpoint handles both JSON and form-urlencoded
- [ ] Using @ai-sdk/gateway (recommended) or direct provider SDK with appropriate API key
