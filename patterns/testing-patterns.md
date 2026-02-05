# Testing Patterns for Slack Agents

This document provides detailed testing patterns for Slack agent projects.

## Test File Organization

```
server/
├── __tests__/
│   ├── setup.ts              # Global test setup and mocks
│   └── helpers/
│       ├── mock-context.ts   # Shared context mocks
│       └── mock-slack.ts     # Slack API mocks
├── lib/
│   └── ai/
│       ├── agent.ts
│       ├── agent.test.ts     # Unit tests (co-located)
│       ├── tools.ts
│       └── tools.test.ts
└── listeners/
    └── assistant/
        ├── thread-started.ts
        └── thread-started.test.ts
```

## Unit Testing Tools

### Testing a Tool Definition

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { getChannelMessages } from './tools';

// Mock the Slack Web API
vi.mock('@slack/web-api', () => ({
  WebClient: vi.fn().mockImplementation(() => ({
    conversations: {
      history: vi.fn().mockResolvedValue({
        ok: true,
        messages: [
          { text: 'Hello', user: 'U123', ts: '123.456' },
          { text: 'World', user: 'U456', ts: '123.457' },
        ],
      }),
    },
  })),
}));

describe('getChannelMessages', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should fetch messages from channel', async () => {
    const result = await getChannelMessages.execute({
      channel_id: 'C12345678',
      limit: 10,
    });

    expect(result.success).toBe(true);
    expect(result.messages).toHaveLength(2);
    expect(result.messages[0].text).toBe('Hello');
  });

  it('should handle empty channel', async () => {
    // Override mock for this test
    vi.mocked(WebClient).mockImplementationOnce(() => ({
      conversations: {
        history: vi.fn().mockResolvedValue({
          ok: true,
          messages: [],
        }),
      },
    }));

    const result = await getChannelMessages.execute({
      channel_id: 'C_EMPTY',
      limit: 10,
    });

    expect(result.success).toBe(true);
    expect(result.messages).toHaveLength(0);
  });

  it('should handle API errors gracefully', async () => {
    vi.mocked(WebClient).mockImplementationOnce(() => ({
      conversations: {
        history: vi.fn().mockRejectedValue(new Error('channel_not_found')),
      },
    }));

    const result = await getChannelMessages.execute({
      channel_id: 'C_INVALID',
      limit: 10,
    });

    expect(result.success).toBe(false);
    expect(result.error).toContain('channel_not_found');
  });
});
```

### Testing the Agent

```typescript
import { describe, it, expect, vi } from 'vitest';
import { createSlackAgent } from './agent';

describe('createSlackAgent', () => {
  const mockContext = {
    channel_id: 'C12345678',
    dm_channel: 'D12345678',
    thread_ts: '1234567890.123456',
    is_dm: false,
    team_id: 'T12345678',
  };

  it('should create agent with required properties', () => {
    const agent = createSlackAgent(mockContext);

    expect(agent).toBeDefined();
    expect(agent.model).toBeDefined();
    expect(agent.tools).toBeDefined();
    expect(agent.system).toBeDefined();
  });

  it('should include context in system prompt', () => {
    const agent = createSlackAgent(mockContext);

    expect(agent.system).toContain('C12345678');
  });

  it('should include all required tools', () => {
    const agent = createSlackAgent(mockContext);
    const toolNames = Object.keys(agent.tools);

    expect(toolNames).toContain('getChannelMessages');
    expect(toolNames).toContain('getThreadMessages');
    expect(toolNames).toContain('joinChannel');
    expect(toolNames).toContain('searchChannels');
  });

  it('should handle DM context', () => {
    const dmContext = {
      ...mockContext,
      is_dm: true,
      channel_id: '',
    };

    const agent = createSlackAgent(dmContext);

    expect(agent).toBeDefined();
    expect(agent.system).toContain('direct message');
  });
});
```

## Testing Event Listeners

### Testing a Message Listener

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { handleAppMention } from './app-mention';

describe('handleAppMention', () => {
  const mockSay = vi.fn();
  const mockClient = {
    chat: {
      postMessage: vi.fn(),
      update: vi.fn(),
    },
  };

  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should respond to mention', async () => {
    const event = {
      type: 'app_mention',
      user: 'U12345678',
      text: '<@BOT123> hello',
      channel: 'C12345678',
      ts: '123.456',
    };

    await handleAppMention({
      event,
      say: mockSay,
      client: mockClient,
    });

    expect(mockSay).toHaveBeenCalled();
  });

  it('should handle mention in thread', async () => {
    const event = {
      type: 'app_mention',
      user: 'U12345678',
      text: '<@BOT123> help',
      channel: 'C12345678',
      ts: '123.456',
      thread_ts: '123.400',
    };

    await handleAppMention({
      event,
      say: mockSay,
      client: mockClient,
    });

    expect(mockSay).toHaveBeenCalledWith(
      expect.objectContaining({
        thread_ts: '123.400',
      })
    );
  });
});
```

### Testing a Slash Command

```typescript
import { describe, it, expect, vi } from 'vitest';
import { handleSampleCommand } from './sample-command';

describe('/sample-command', () => {
  const mockAck = vi.fn();
  const mockRespond = vi.fn();

  it('should acknowledge command immediately', async () => {
    await handleSampleCommand({
      command: {
        command: '/sample-command',
        text: 'test input',
        user_id: 'U12345678',
        channel_id: 'C12345678',
      },
      ack: mockAck,
      respond: mockRespond,
    });

    expect(mockAck).toHaveBeenCalled();
  });

  it('should respond with expected format', async () => {
    await handleSampleCommand({
      command: {
        command: '/sample-command',
        text: 'test',
        user_id: 'U12345678',
        channel_id: 'C12345678',
      },
      ack: mockAck,
      respond: mockRespond,
    });

    expect(mockRespond).toHaveBeenCalledWith(
      expect.objectContaining({
        response_type: 'ephemeral',
      })
    );
  });
});
```

## E2E Testing Patterns

### Testing Full Message Flow

```typescript
// server/__tests__/e2e/message-flow.e2e.test.ts
import { describe, it, expect, vi, beforeAll, afterAll } from 'vitest';
import { createTestApp } from '../helpers/test-app';

describe('E2E: Message Flow', () => {
  let app;

  beforeAll(async () => {
    app = await createTestApp();
  });

  afterAll(async () => {
    await app.stop();
  });

  it('should handle complete mention flow', async () => {
    const response = await app.simulateEvent('app_mention', {
      user: 'U12345678',
      text: '<@BOT123> what channels am I in?',
      channel: 'C12345678',
    });

    expect(response.messages).toHaveLength(1);
    expect(response.messages[0].text).toContain('channel');
  });

  it('should handle conversation in thread', async () => {
    // First message
    const initial = await app.simulateEvent('app_mention', {
      user: 'U12345678',
      text: '<@BOT123> start a task',
      channel: 'C12345678',
      ts: '100.001',
    });

    // Follow-up in thread
    const followUp = await app.simulateEvent('message', {
      user: 'U12345678',
      text: 'continue please',
      channel: 'C12345678',
      thread_ts: '100.001',
    });

    expect(followUp.messages[0].thread_ts).toBe('100.001');
  });
});
```

## Mock Helpers

### Creating a Mock Slack Context

```typescript
// server/__tests__/helpers/mock-context.ts
export function createMockContext(overrides = {}) {
  return {
    channel_id: 'C12345678',
    dm_channel: 'D12345678',
    thread_ts: undefined,
    is_dm: false,
    team_id: 'T12345678',
    user_id: 'U12345678',
    ...overrides,
  };
}

export function createMockThreadContext(threadTs: string) {
  return createMockContext({
    thread_ts: threadTs,
  });
}

export function createMockDMContext() {
  return createMockContext({
    is_dm: true,
    channel_id: 'D12345678',
  });
}
```

### Creating a Mock Slack Client

```typescript
// server/__tests__/helpers/mock-slack.ts
import { vi } from 'vitest';

export function createMockSlackClient() {
  return {
    conversations: {
      history: vi.fn().mockResolvedValue({ ok: true, messages: [] }),
      replies: vi.fn().mockResolvedValue({ ok: true, messages: [] }),
      join: vi.fn().mockResolvedValue({ ok: true }),
      list: vi.fn().mockResolvedValue({ ok: true, channels: [] }),
      info: vi.fn().mockResolvedValue({ ok: true, channel: {} }),
    },
    chat: {
      postMessage: vi.fn().mockResolvedValue({ ok: true, ts: '123.456' }),
      update: vi.fn().mockResolvedValue({ ok: true }),
      delete: vi.fn().mockResolvedValue({ ok: true }),
    },
    users: {
      info: vi.fn().mockResolvedValue({ ok: true, user: {} }),
    },
    reactions: {
      add: vi.fn().mockResolvedValue({ ok: true }),
      remove: vi.fn().mockResolvedValue({ ok: true }),
    },
  };
}
```

## Test Coverage Guidelines

Aim for these coverage targets:

| Category | Target |
|----------|--------|
| Tools | 90%+ |
| Agent logic | 85%+ |
| Event listeners | 80%+ |
| Utilities | 90%+ |
| Overall | 80%+ |

Run coverage report:
```bash
pnpm test:coverage
```
