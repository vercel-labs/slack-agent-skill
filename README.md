# Slack Agent Skill

An agent-agnostic skill for building and deploying Slack agents on Vercel. This skill provides comprehensive guidance for developing Slack bots with AI capabilities using the [Vercel Slack Agent Template](https://github.com/vercel-partner-solutions/slack-agent-template).

## Features

- **Interactive Setup Wizard**: Step-by-step guidance from project creation to production deployment
- **Quality Standards**: Embedded testing and code quality requirements
- **AI Integration**: Support for Vercel AI Gateway and direct provider SDKs
- **Comprehensive Patterns**: Slack-specific development patterns and best practices
- **Testing Framework**: Vitest configuration and sample tests

## Installation

### Via skills.sh (Recommended)

Visit skills.sh https://skills.sh/ and search for  cls-audit , or install directly:

npx skills add vercel-labs/slack-agent-skill

### Manual Installation

Clone the repository into your Claude Code skills directory:

git clone https://github.com/vercel-labs/slack-agent-skill.git ~/.claude/skills/slack-agent-skill

## Usage

### Starting a New Project

Run the slash command:

```
/slack-agent

Or with arguments:
/slack-agent new       # Start fresh project
/slack-agent configure # Configure existing project
/slack-agent deploy    # Deploy to production
/slack-agent test      # Set up testing
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

## Key Commands

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

## Related Resources

- [Vercel Slack Agent Template](https://github.com/vercel-partner-solutions/slack-agent-template)
- [AI SDK Documentation](https://ai-sdk.dev)
- [Slack Bolt Documentation](https://slack.dev/bolt-js)
- [Vercel Documentation](https://vercel.com/docs)

## License

Apache 2.0 - See [LICENSE](LICENSE) for details.
