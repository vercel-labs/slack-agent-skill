# Slack Agent Skill

An agent-agnostic skill for building and deploying Slack agents on Vercel. This skill provides comprehensive guidance for developing Slack bots with AI capabilities using the [Vercel Slack Agent Template](https://github.com/vercel-partner-solutions/slack-agent-template).

## Features

- **Interactive Setup Wizard**: Step-by-step guidance from project creation to production deployment
- **Quality Standards**: Embedded testing and code quality requirements
- **AI Integration**: Support for Vercel AI Gateway and direct provider SDKs
- **Comprehensive Patterns**: Slack-specific development patterns and best practices
- **Testing Framework**: Vitest configuration and sample tests

## Installation

### For Claude Code

Add this skill to your Claude Code configuration:

```bash
# Clone the skill repository
git clone https://github.com/vercel-labs/slack-agent-skill.git ~/.claude/skills/slack-agent-skill
```

Then reference it in your project's `.claude/settings.json`:

```json
{
  "skills": [
    "~/.claude/skills/slack-agent-skill/SKILL.md"
  ]
}
```

### Manual Reference

You can also reference specific files directly when working with an AI assistant:

- `SKILL.md` - Main development skill with quality standards
- `wizard/README.md` - Interactive setup wizard
- `patterns/` - Development patterns for testing and Slack APIs
- `reference/` - Technical reference documentation

## Repository Structure

```
slack-agent-skill/
├── SKILL.md                           # Main skill (development standards)
├── wizard/                            # Interactive setup wizard
│   ├── README.md                      # Wizard overview
│   ├── 1-project-setup.md             # Clone template, choose LLM
│   ├── 2-create-slack-app.md          # Customize manifest, create app
│   ├── 3-configure-environment.md     # Set up .env
│   ├── 4-test-locally.md              # Dev server + ngrok
│   ├── 5-deploy-production.md         # Vercel deployment
│   └── 6-setup-testing.md             # Vitest configuration
├── patterns/                          # Development patterns
│   ├── testing-patterns.md            # Unit/E2E testing
│   └── slack-patterns.md              # Slack Block Kit & APIs
├── reference/                         # Technical reference
│   ├── env-vars.md                    # Environment variables
│   ├── ai-sdk.md                      # AI SDK integration
│   ├── slack-setup.md                 # Slack app setup
│   └── vercel-setup.md                # Vercel deployment
└── templates/                         # Code templates
    ├── vitest.config.ts               # Vitest configuration
    ├── test-setup.ts                  # Test utilities
    └── sample-tests/                  # Example tests
        ├── agent.test.ts
        └── tools.test.ts
```

## Usage

### Starting a New Project

Ask your AI assistant to run the setup wizard:

```
Help me create a new Slack agent using the wizard
```

The wizard will guide you through:
1. Project setup and LLM provider selection
2. Slack app creation with customized manifest
3. Environment configuration
4. Local testing with ngrok
5. Production deployment to Vercel
6. Test framework setup

### Development

When working on an existing Slack agent project, the skill automatically provides:

- Code quality standards (linting, testing, TypeScript)
- Slack-specific patterns (events, slash commands, Block Kit)
- AI integration guidance (Vercel AI Gateway, direct providers)
- Deployment best practices

### Key Commands

```bash
# Development
pnpm dev              # Start local dev server
ngrok http 3000       # Expose local server

# Quality
pnpm lint             # Check linting
pnpm lint --write     # Auto-fix lint issues
pnpm typecheck        # TypeScript check
pnpm test             # Run tests

# Deployment
vercel                # Deploy to Vercel
vercel --prod         # Production deployment
```

## Quality Standards

The skill enforces these requirements:

- **Unit tests** for all exported functions
- **E2E tests** for user-facing changes
- **Linting** must pass (Biome)
- **TypeScript** must compile without errors
- **All tests** must pass before completion

## AI Integration Options

### Vercel AI Gateway (Recommended)

No API keys needed when deployed on Vercel:

```typescript
import { generateText } from "ai";
import { gateway } from "@ai-sdk/gateway";

const result = await generateText({
  model: gateway("openai/gpt-4o-mini"),
  prompt: "Your prompt",
});
```

### Direct Provider SDKs

For more control or non-Vercel deployments:

```typescript
import { openai } from "@ai-sdk/openai";
import { anthropic } from "@ai-sdk/anthropic";
import { google } from "@ai-sdk/google";
```

## Related Resources

- [Vercel Slack Agent Template](https://github.com/vercel-partner-solutions/slack-agent-template)
- [AI SDK Documentation](https://ai-sdk.dev)
- [Slack Bolt Documentation](https://slack.dev/bolt-js)
- [Vercel Documentation](https://vercel.com/docs)

## License

Apache 2.0 - See [LICENSE](LICENSE) for details.
