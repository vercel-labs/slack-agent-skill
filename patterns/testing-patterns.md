# Testing Patterns for Slack Agents

This document provides detailed testing patterns for Slack agent projects built with the Chat SDK.

## Test File Organization

```
lib/
├── __tests__/
│   ├── setup.ts              # Global test setup and mocks
│   └── helpers/
│       ├── mock-context.ts   # Shared context mocks
│       └── mock-thread.ts    # Chat SDK thread mocks
├── bot.tsx                   # Bot instance
├── bot.test.ts               # Bot handler tests
├── ai/
│   ├── agent.ts
│   ├── agent.test.ts         # Unit tests (co-located)
│   └── tools/
│       ├── search.ts
│       └── search.test.ts
app/
├── api/
│   └── webhooks/
│       └── [platform]/
│           └── route.ts
```

## Unit Testing Tools

### Testing a Tool Definition

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { getChannelMessages } from './tools';

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
    const result = await getChannelMessages.execute({
      channel_id: 'C_EMPTY',
      limit: 10,
    });

    expect(result.success).toBe(true);
    expect(result.messages).toHaveLength(0);
  });

  it('should handle API errors gracefully', async () => {
    const result = await getChannelMessages.execute({
      channel_id: 'C_INVALID',
      limit: 10,
    });

    expect(result.success).toBe(false);
    expect(result.error).toContain('channel_not_found');
  });
});
```

### Testing Bot Handlers

```typescript
import { describe, it, expect, vi } from 'vitest';
import { createMockThread, createMockMessage } from './helpers/mock-thread';

describe('bot.onNewMention', () => {
  it('should respond to mention and subscribe', async () => {
    const thread = createMockThread();
    const message = createMockMessage({ text: 'hello' });

    // Import and call your handler
    await handleMention(thread, message);

    expect(thread.subscribe).toHaveBeenCalled();
    expect(thread.post).toHaveBeenCalled();
  });

  it('should handle mention in thread', async () => {
    const thread = createMockThread({ threadTs: '123.400' });
    const message = createMockMessage({ text: 'help' });

    await handleMention(thread, message);

    expect(thread.post).toHaveBeenCalled();
  });
});
```

## Testing Event Handlers

### Testing a Slash Command Handler

```typescript
import { describe, it, expect, vi } from 'vitest';
import { createMockSlashCommandEvent } from './helpers/mock-thread';

describe('/sample-command', () => {
  it('should process command and respond', async () => {
    const event = createMockSlashCommandEvent({
      text: 'test input',
      userId: 'U12345678',
    });

    await handleSampleCommand(event);

    expect(event.thread.post).toHaveBeenCalledWith(
      expect.stringContaining('Result')
    );
  });

  it('should handle errors gracefully', async () => {
    const event = createMockSlashCommandEvent({
      text: '',
      userId: 'U12345678',
    });

    await handleSampleCommand(event);

    expect(event.thread.post).toHaveBeenCalledWith(
      expect.stringContaining('went wrong')
    );
  });
});
```

### Testing an Action Handler

```typescript
import { describe, it, expect, vi } from 'vitest';
import { createMockActionEvent } from './helpers/mock-thread';

describe('button_click action', () => {
  it('should handle button click', async () => {
    const event = createMockActionEvent({
      actionId: 'button_click',
      value: 'clicked_value',
    });

    await handleButtonClick(event);

    expect(event.thread.post).toHaveBeenCalled();
  });
});
```

## E2E Testing Patterns

### Testing Full Message Flow

```typescript
import { describe, it, expect, vi, beforeAll, afterAll } from 'vitest';

describe('E2E: Message Flow', () => {
  it('should handle complete mention flow', async () => {
    const thread = createMockThread();
    const message = createMockMessage({
      text: 'what channels am I in?',
    });

    await handleMention(thread, message);

    expect(thread.subscribe).toHaveBeenCalled();
    expect(thread.post).toHaveBeenCalled();
  });

  it('should handle conversation in thread', async () => {
    const thread = createMockThread({ threadTs: '100.001' });

    // First message (mention)
    const mention = createMockMessage({ text: 'start a task' });
    await handleMention(thread, mention);

    // Follow-up (subscribed message)
    const followUp = createMockMessage({ text: 'continue please' });
    await handleSubscribedMessage(thread, followUp);

    expect(thread.post).toHaveBeenCalledTimes(2);
  });
});
```

## Mock Helpers

### Creating a Mock Thread

```typescript
// lib/__tests__/helpers/mock-thread.ts
import { vi } from 'vitest';

export function createMockThread(overrides = {}) {
  return {
    post: vi.fn().mockResolvedValue(undefined),
    subscribe: vi.fn().mockResolvedValue(undefined),
    startTyping: vi.fn().mockResolvedValue(undefined),
    state: {
      get: vi.fn().mockResolvedValue(null),
      set: vi.fn().mockResolvedValue(undefined),
    },
    channelId: 'C12345678',
    threadTs: undefined,
    ...overrides,
  };
}

export function createMockMessage(overrides = {}) {
  return {
    text: 'test message',
    userId: 'U12345678',
    ts: '1234567890.123456',
    ...overrides,
  };
}

export function createMockSlashCommandEvent(overrides = {}) {
  return {
    text: '',
    userId: 'U12345678',
    channelId: 'C12345678',
    thread: createMockThread(),
    openModal: vi.fn().mockResolvedValue(undefined),
    ...overrides,
  };
}

export function createMockActionEvent(overrides = {}) {
  return {
    actionId: '',
    value: '',
    userId: 'U12345678',
    thread: createMockThread(),
    ...overrides,
  };
}
```

### Creating a Mock Context

```typescript
// lib/__tests__/helpers/mock-context.ts
export function createMockContext(overrides = {}) {
  return {
    channel_id: 'C12345678',
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
