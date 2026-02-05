# Phase 1: Project Setup

This phase handles creating a new Slack agent project from the template.

## Check Current State

First, look for existing project indicators:
- `package.json` with `@slack/bolt` dependency
- `manifest.json` (Slack app manifest)
- `server/` directory with agent code
- `.env` file with credentials

If these exist, this is an existing project - skip to Phase 2 or 3.

---

## For NEW Projects

### Step 1.1: Understand the Agent Purpose

Before cloning the template, ask the user what they're building:

> **What kind of Slack agent are you building?**
>
> Examples: joke bot, support assistant, weather bot, task manager, standup bot, etc.

Based on their response, generate a recommended project name:
- "joke bot" -> `joke-slack-agent`
- "customer support" -> `support-slack-agent`
- "weather" -> `weather-slack-agent`
- "task manager" -> `taskbot-slack-agent`
- "standup" -> `standup-slack-agent`

Use the pattern: `<purpose>-slack-agent` (lowercase, hyphens, no spaces).

**Store this context** - you'll use it for manifest customization in Phase 2.

### Step 1.2: Choose LLM Provider (if using AI)

Ask the user if their agent will use AI/LLM capabilities:

> **Will your agent use AI/LLM capabilities?**
>
> 1. **Yes, using Vercel AI Gateway** (Recommended)
>    - Easiest setup - no API keys needed when deployed on Vercel
>    - Access to multiple providers (OpenAI, Anthropic, Google, etc.)
>    - Built-in rate limiting and observability
>
> 2. **Yes, using a direct provider SDK**
>    - Direct integration with OpenAI, Anthropic, or other providers
>    - Requires managing your own API keys
>    - More control over provider-specific features
>
> 3. **No LLM needed**
>    - Simple automation bot (slash commands, integrations, etc.)
>    - Can add AI capabilities later

Based on their choice:

**If Vercel AI Gateway (default):**
- The template already includes `ai` and `@ai-sdk/gateway` packages
- No additional setup needed - works automatically on Vercel
- Store this choice for Phase 3 (no AI keys needed in .env)

**If Direct Provider SDK:**
- Ask which provider: OpenAI, Anthropic, Google, or other
- Note they'll need to add the provider package (e.g., `@ai-sdk/openai`)
- Store this choice for Phase 3 (will need API key in .env)

**If No LLM:**
- Skip AI-related configuration
- The base Slack functionality will work without AI packages

**Store this context** - you'll use it for environment configuration in Phase 3.

### Step 1.3: Clone the Template

Clone the official Slack Agent Template with the recommended name:

```bash
# Recommended name based on your agent: <recommended-name>
git clone https://github.com/vercel-partner-solutions/slack-agent-template <recommended-name>
cd <recommended-name>
pnpm install

# Start fresh git history
rm -rf .git
git init
git add .
git commit -m "Initial commit from slack-agent-template"
```

Ask the user to confirm or customize the project name before proceeding.

---

## For EXISTING Projects

Verify the project structure:
- `server/api/slack/events.post.ts` - Events endpoint
- `server/lib/ai/agent.ts` - Agent logic
- `server/lib/ai/tools.ts` - Tool definitions
- `server/listeners/` - Event handlers
- `manifest.json` - Slack app configuration

If the structure looks correct, proceed to Phase 2 (Create Slack App) or Phase 3 (Configure Environment) depending on what's already done.

---

## Next Phase

Once the project is set up, proceed to [Phase 2: Create Slack App](./2-create-slack-app.md).
