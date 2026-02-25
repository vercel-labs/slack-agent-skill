# Phase 6: Set Up Testing (Optional but Recommended)

This phase guides the user through setting up a testing framework for their Slack agent.

---

## Step 6.1: Install Test Dependencies

```bash
pnpm add -D vitest @vitest/coverage-v8
```

---

## Step 6.2: Create Test Config

Create `vitest.config.ts` in the project root with the following content:

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // Use globals (describe, it, expect) without imports
    globals: true,

    // Node environment for server-side code
    environment: 'node',

    // Test file patterns
    include: [
      'lib/**/*.test.ts',
      'lib/**/*.test.tsx',
      'app/**/*.test.ts',
    ],

    // Exclude patterns
    exclude: [
      'node_modules',
      '.next',
      'dist',
    ],

    // Coverage configuration
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      include: ['lib/**/*.ts', 'lib/**/*.tsx', 'app/**/*.ts'],
      exclude: [
        'lib/**/*.test.ts',
        'lib/**/*.test.tsx',
        'lib/**/*.d.ts',
        'lib/__tests__/**',
      ],
      // Coverage thresholds (adjust as needed)
      thresholds: {
        lines: 70,
        functions: 70,
        branches: 60,
        statements: 70,
      },
    },

    // Setup files run before each test file
    setupFiles: ['./lib/__tests__/setup.ts'],

    // Timeout for async operations (ms)
    testTimeout: 10000,

    // Retry failed tests (useful for flaky network tests)
    retry: 0,

    // Reporter configuration
    reporters: ['verbose'],
  },
});
```

You can also copy this from `./templates/vitest.config.ts`.

---

## Step 6.3: Create Test Setup

Create `lib/__tests__/setup.ts` with test utilities and mocks. You can copy the template from `./templates/test-setup.ts`.

This setup file provides:
- Environment variable stubs for tests
- Chat SDK mocking (thread, message, state)
- Mock factories for creating test fixtures
- Test lifecycle hooks

---

## Step 6.4: Add Test Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "typecheck": "tsc --noEmit"
  }
}
```

---

## Step 6.5: Create Sample Tests

Copy the sample test templates from `./templates/sample-tests/` to your project:

- `agent.test.ts` - Sample bot handler tests
- `tools.test.ts` - Sample tool unit tests

Customize these templates for your specific implementation.

---

## Step 6.6: Run Tests

```bash
pnpm test
```

---

## Test Coverage Guidelines

Aim for these coverage targets:

| Category | Target |
|----------|--------|
| Tools | 90%+ |
| Agent logic | 85%+ |
| Event handlers | 80%+ |
| Utilities | 90%+ |
| Overall | 80%+ |

Run coverage report:
```bash
pnpm test:coverage
```

---

## Security Reminders

- NEVER commit `.env` files
- NEVER log full API tokens
- Use different Slack apps for dev and production
- Rotate credentials if exposed

---

## Complete!

Your Slack agent is now set up with:
- Next.js project with Chat SDK
- Slack app with customized manifest
- Environment configuration
- Local development workflow
- Production deployment on Vercel
- Testing infrastructure

For ongoing development, refer to:
- `./SKILL.md` - Development standards and patterns
- `./patterns/testing-patterns.md` - Detailed testing guidance
- `./patterns/slack-patterns.md` - Slack-specific patterns
- `./reference/` - Technical reference documentation
