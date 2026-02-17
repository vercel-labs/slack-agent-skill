# Slack Development Patterns

This document covers Slack-specific patterns and best practices for building agents.

## Block Kit UI

### Basic Message with Blocks

```typescript
import { WebClient } from '@slack/web-api';

const client = new WebClient(process.env.SLACK_BOT_TOKEN);

await client.chat.postMessage({
  channel: channelId,
  text: 'Fallback text for notifications', // Required for accessibility
  blocks: [
    {
      type: 'section',
      text: {
        type: 'mrkdwn',
        text: '*Hello!* This is a formatted message.',
      },
    },
    {
      type: 'divider',
    },
    {
      type: 'section',
      text: {
        type: 'mrkdwn',
        text: 'Choose an option:',
      },
      accessory: {
        type: 'button',
        text: {
          type: 'plain_text',
          text: 'Click Me',
        },
        action_id: 'button_click',
        value: 'button_value',
      },
    },
  ],
});
```

### Interactive Actions

```typescript
// Button with confirmation
{
  type: 'button',
  text: { type: 'plain_text', text: 'Delete' },
  style: 'danger',
  action_id: 'delete_item',
  value: itemId,
  confirm: {
    title: { type: 'plain_text', text: 'Confirm Delete' },
    text: { type: 'mrkdwn', text: 'Are you sure you want to delete this?' },
    confirm: { type: 'plain_text', text: 'Delete' },
    deny: { type: 'plain_text', text: 'Cancel' },
  },
}

// Select menu
{
  type: 'static_select',
  placeholder: { type: 'plain_text', text: 'Select an option' },
  action_id: 'select_option',
  options: [
    { text: { type: 'plain_text', text: 'Option 1' }, value: 'opt1' },
    { text: { type: 'plain_text', text: 'Option 2' }, value: 'opt2' },
  ],
}
```

### Context and Header Blocks

```typescript
// Header
{
  type: 'header',
  text: { type: 'plain_text', text: 'Task Summary' },
}

// Context (small text, often for metadata)
{
  type: 'context',
  elements: [
    { type: 'mrkdwn', text: 'Created by <@U12345678>' },
    { type: 'mrkdwn', text: '|' },
    { type: 'mrkdwn', text: '<!date^1234567890^{date_short}|Jan 1, 2024>' },
  ],
}
```

## Message Formatting (mrkdwn)

Slack uses its own markdown variant called mrkdwn.

### Text Formatting
```
*bold text*
_italic text_
~strikethrough~
`inline code`
```code block```
> blockquote
```

### Links and Mentions
```
<https://example.com|Link Text>
<@U12345678>              # User mention
<#C12345678>              # Channel link
<!here>                   # @here mention
<!channel>                # @channel mention
<!date^1234567890^{date_short}|fallback>  # Date formatting
```

### Lists
```
Slack doesn't support markdown lists, use:
* Bullet point (use the actual bullet character)
1. Numbered manually
```

## Events Endpoint

Use `@vercel/slack-bolt` for all Slack event handling. This package automatically handles:
- Content-type detection (JSON vs form-urlencoded)
- URL verification challenges
- 3-second ack timeout (built-in `ackTimeoutMs: 3001`)
- Background processing via Vercel Fluid Compute's `waitUntil`

### Bolt App Setup

```typescript
// server/bolt/app.ts
import { App } from "@slack/bolt";
import { VercelReceiver } from "@vercel/slack-bolt";

const receiver = new VercelReceiver();
const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET,
  receiver,
  deferInitialization: true,
});

export { app, receiver };
```

### Events Handler

```typescript
// server/api/slack/events.post.ts
import { createHandler } from "@vercel/slack-bolt";
import { eventHandler, toWebRequest } from "h3";
import { app, receiver } from "../../bolt/app";

const handler = createHandler(app, receiver);
export default eventHandler(async (event) => handler(toWebRequest(event)));
```

### Content Type Reference

`@vercel/slack-bolt` automatically handles all content types:

| Event Type | Content-Type | Handled Automatically |
|------------|--------------|----------------------|
| Slash commands | `application/x-www-form-urlencoded` | ✅ |
| Events API | `application/json` | ✅ |
| Interactivity | `application/json` | ✅ |
| URL verification | `application/json` | ✅ |

## Event Handling Patterns

### App Mention Handler

```typescript
// server/listeners/events/app-mention.ts
import type { App } from '@slack/bolt';

export function registerAppMention(app: App) {
  app.event('app_mention', async ({ event, client, say }) => {
    try {
      // Extract the actual message (remove bot mention)
      const text = event.text.replace(/<@[A-Z0-9]+>/g, '').trim();

      // Respond in thread if it's a thread, otherwise start new thread
      const thread_ts = event.thread_ts || event.ts;

      await say({
        text: `Processing your request: "${text}"`,
        thread_ts,
      });

      // Process with agent...
    } catch (error) {
      console.error('Error handling mention:', error);
      await say({
        text: 'Sorry, I encountered an error processing your request.',
        thread_ts: event.thread_ts || event.ts,
      });
    }
  });
}
```

### Message Handler with Filtering

```typescript
// Only respond to messages that aren't from bots
app.message(async ({ message, say }) => {
  // Skip bot messages
  if ('bot_id' in message) return;

  // Skip message edits
  if ('subtype' in message && message.subtype === 'message_changed') return;

  // Only respond in DMs or when in a thread
  if (message.channel_type !== 'im' && !message.thread_ts) return;

  // Handle the message...
});
```

### Slash Command Handler

```typescript
// server/listeners/commands/sample-command.ts
import type { App } from '@slack/bolt';

export function registerSampleCommand(app: App) {
  app.command('/sample-command', async ({ command, ack, respond }) => {
    // Always acknowledge within 3 seconds
    await ack();

    try {
      const { text, user_id, channel_id } = command;

      // Process command...
      const result = await processCommand(text);

      // Respond (ephemeral by default)
      await respond({
        response_type: 'ephemeral', // or 'in_channel'
        text: `Result: ${result}`,
      });
    } catch (error) {
      await respond({
        response_type: 'ephemeral',
        text: 'Sorry, something went wrong.',
      });
    }
  });
}
```

**Sync vs Async Pattern:**
- **Sync**: `await ack({ text: "Response" })` - immediate response in ack
- **Async**: `await ack()` then `await respond({...})` - deferred response

For async commands doing AI processing, always use the async pattern. When using `@vercel/slack-bolt`, the response handling is automatic—no manual empty response needed.

## Action Handlers

### Button Click Handler

```typescript
app.action('button_click', async ({ body, ack, client }) => {
  await ack();

  const { user, channel, message, actions } = body;
  const buttonValue = actions[0].value;

  // Update the original message
  await client.chat.update({
    channel: channel.id,
    ts: message.ts,
    text: 'Updated message',
    blocks: [/* new blocks */],
  });
});
```

### Select Menu Handler

```typescript
app.action('select_option', async ({ body, ack, client }) => {
  await ack();

  const selectedValue = body.actions[0].selected_option.value;

  // Handle selection...
});
```

## Modal Patterns

### Opening a Modal

```typescript
app.shortcut('open_modal', async ({ shortcut, ack, client }) => {
  await ack();

  await client.views.open({
    trigger_id: shortcut.trigger_id,
    view: {
      type: 'modal',
      callback_id: 'modal_submit',
      title: { type: 'plain_text', text: 'My Modal' },
      submit: { type: 'plain_text', text: 'Submit' },
      close: { type: 'plain_text', text: 'Cancel' },
      blocks: [
        {
          type: 'input',
          block_id: 'input_block',
          label: { type: 'plain_text', text: 'Your Input' },
          element: {
            type: 'plain_text_input',
            action_id: 'input_value',
            placeholder: { type: 'plain_text', text: 'Enter something...' },
          },
        },
      ],
    },
  });
});
```

### Handling Modal Submission

```typescript
app.view('modal_submit', async ({ ack, body, view, client }) => {
  // Validate input
  const inputValue = view.state.values.input_block.input_value.value;

  if (!inputValue || inputValue.length < 3) {
    await ack({
      response_action: 'errors',
      errors: {
        input_block: 'Please enter at least 3 characters',
      },
    });
    return;
  }

  await ack();

  // Process the submission...
  await client.chat.postMessage({
    channel: body.user.id,
    text: `You submitted: ${inputValue}`,
  });
});
```

## Error Handling

### Graceful Error Responses

```typescript
async function handleWithErrorRecovery(
  operation: () => Promise<void>,
  say: SayFn,
  thread_ts?: string
) {
  try {
    await operation();
  } catch (error) {
    console.error('Operation failed:', error);

    let userMessage = 'Something went wrong. Please try again.';

    if (error instanceof SlackAPIError) {
      if (error.code === 'channel_not_found') {
        userMessage = "I don't have access to that channel.";
      } else if (error.code === 'not_in_channel') {
        userMessage = 'Please invite me to the channel first.';
      }
    }

    await say({
      text: userMessage,
      thread_ts,
    });
  }
}
```

### Rate Limiting

```typescript
import pRetry from 'p-retry';

async function sendMessageWithRetry(client: WebClient, options: ChatPostMessageArguments) {
  return pRetry(
    () => client.chat.postMessage(options),
    {
      retries: 3,
      onFailedAttempt: (error) => {
        if (error.code === 'rate_limited') {
          const retryAfter = error.retryAfter || 1;
          console.log(`Rate limited. Retrying after ${retryAfter}s`);
        }
      },
    }
  );
}
```

## Thread Management

### Maintaining Thread Context

```typescript
// Always reply in the same thread
const thread_ts = event.thread_ts || event.ts;

await say({
  text: 'Response message',
  thread_ts,
});
```

### Broadcasting Thread Replies

```typescript
// Reply in thread AND post to channel
await client.chat.postMessage({
  channel: channelId,
  thread_ts: parentTs,
  text: 'Important update!',
  reply_broadcast: true, // Also posts to channel
});
```

## Typing Indicators

### Showing Typing Status

```typescript
// For Assistant threads
app.event('assistant_thread_started', async ({ event, client }) => {
  // Set typing status
  await client.assistant.threads.setStatus({
    channel_id: event.channel,
    thread_ts: event.thread_ts,
    status: 'is thinking...',
  });

  // Process...

  // Clear status (happens automatically when you respond)
});
```

## Best Practices Summary

1. **Always acknowledge** within 3 seconds for interactive elements
2. **Use threads** for conversations to keep channels clean
3. **Provide fallback text** in all Block Kit messages
4. **Handle errors gracefully** with user-friendly messages
5. **Respect rate limits** with exponential backoff
6. **Validate inputs** before processing
7. **Use ephemeral messages** for sensitive or temporary information
8. **Log errors** with context for debugging
