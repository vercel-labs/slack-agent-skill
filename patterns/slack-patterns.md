# Slack Development Patterns

This document covers Slack-specific patterns and best practices for building agents with the Chat SDK.

## Rich UI with JSX Components

### Basic Message with Card

Chat SDK uses JSX components instead of raw Block Kit JSON. Files using JSX must have the `.tsx` extension.

```tsx
import { Card, CardText as Text, Actions, Button, Divider } from "chat";

await thread.post(
  <Card title="Hello!">
    <Text>*Hello!* This is a formatted message.</Text>
    <Divider />
    <Text>Choose an option:</Text>
    <Actions>
      <Button id="button_click" value="button_value" style="primary">Click Me</Button>
    </Actions>
  </Card>
);
```

### Interactive Actions

```tsx
import { Card, CardText as Text, Actions, Button } from "chat";

// Button with danger style
await thread.post(
  <Card>
    <Text>Are you sure you want to delete this?</Text>
    <Actions>
      <Button id="delete_item" value={itemId} style="danger">Delete</Button>
      <Button id="cancel">Cancel</Button>
    </Actions>
  </Card>
);

// Select menu
import { Select, Option } from "chat";

await thread.post(
  <Card>
    <Text>Select an option:</Text>
    <Actions>
      <Select id="select_option" placeholder="Choose...">
        <Option value="opt1">Option 1</Option>
        <Option value="opt2">Option 2</Option>
      </Select>
    </Actions>
  </Card>
);
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

## Webhook Route

The Chat SDK webhook route handles all incoming Slack events through a single dynamic endpoint:

```typescript
// app/api/webhooks/[platform]/route.ts
import { after } from "next/server";
import { bot } from "@/lib/bot";

export async function POST(request: Request, context: { params: Promise<{ platform: string }> }) {
  const { platform } = await context.params;
  const handler = bot.webhooks[platform as keyof typeof bot.webhooks];
  if (!handler) return new Response("Unknown platform", { status: 404 });
  return handler(request, { waitUntil: (task) => after(() => task) });
}
```

The Chat SDK automatically handles:
- Content-type detection (JSON vs form-urlencoded)
- URL verification challenges
- Slack's 3-second ack timeout
- Background processing via `waitUntil`
- Signature verification using `SLACK_SIGNING_SECRET`

### Content Type Reference

The Chat SDK handles all content types automatically:

| Event Type | Content-Type | Handled Automatically |
|------------|--------------|----------------------|
| Slash commands | `application/x-www-form-urlencoded` | Yes |
| Events API | `application/json` | Yes |
| Interactivity | `application/json` | Yes |
| URL verification | `application/json` | Yes |

## Event Handling Patterns

### Mention Handler

```typescript
// lib/bot.tsx
bot.onNewMention(async (thread, message) => {
  try {
    // The mention prefix is already stripped from message.text
    const text = message.text;

    // Subscribe to follow-up messages in this thread
    await thread.subscribe();

    await thread.post(`Processing your request: "${text}"`);

    // Process with agent...
  } catch (error) {
    console.error("Error handling mention:", error);
    await thread.post("Sorry, I encountered an error processing your request.");
  }
});
```

### Subscribed Message Handler

```typescript
bot.onSubscribedMessage(async (thread, message) => {
  // Skip bot messages (handled automatically by Chat SDK)
  // Handle follow-up messages in subscribed threads
  await thread.post(`You said: ${message.text}`);
});
```

### Slash Command Handler

```typescript
bot.onSlashCommand("/sample-command", async (event) => {
  const { text } = event;

  try {
    const result = await processCommand(text);

    // Respond to the command
    await event.thread.post(`Result: ${result}`);
  } catch (error) {
    await event.thread.post("Sorry, something went wrong.");
  }
});
```

The Chat SDK handles the 3-second ack timeout and background processing automatically. You don't need fire-and-forget patterns or manual `ack()` calls.

### Long-Running Slash Commands (AI, API calls)

Unlike Bolt, the Chat SDK handles background processing via `waitUntil` automatically. You can simply `await` long-running operations:

```typescript
bot.onSlashCommand("/ai-command", async (event) => {
  // Start typing indicator
  await event.thread.startTyping();

  // This can take as long as needed - Chat SDK handles the ack automatically
  const result = await generateWithAI(event.text);

  // Post the result
  await event.thread.post(result);
});
```

**No fire-and-forget pattern needed.** The Chat SDK acknowledges the request immediately and processes the handler in the background.

## Action Handlers

### Button Click Handler

```typescript
bot.onAction("button_click", async (event) => {
  const value = event.value;

  // Update the thread with new content
  await event.thread.post(`You clicked: ${value}`);
});
```

### Select Menu Handler

```typescript
bot.onAction("select_option", async (event) => {
  const selectedValue = event.value;

  await event.thread.post(`You selected: ${selectedValue}`);
});
```

## Modal Patterns

### Opening a Modal

```tsx
import { Modal, TextInput, Select, Option } from "chat";

bot.onSlashCommand("/open-form", async (event) => {
  await event.openModal(
    <Modal title="My Modal" submitLabel="Submit" callbackId="modal_submit">
      <TextInput id="input_value" label="Your Input" placeholder="Enter something..." />
    </Modal>
  );
});
```

### Handling Modal Submission

```typescript
bot.onAction("modal_submit", async (event) => {
  const inputValue = event.values?.input_value;

  if (!inputValue || inputValue.length < 3) {
    // Return validation errors
    return { errors: { input_value: "Please enter at least 3 characters" } };
  }

  await event.thread.post(`You submitted: ${inputValue}`);
});
```

## Error Handling

### Graceful Error Responses

```typescript
bot.onNewMention(async (thread, message) => {
  try {
    await processMessage(thread, message);
  } catch (error) {
    console.error("Operation failed:", error);

    let userMessage = "Something went wrong. Please try again.";

    if (error instanceof Error) {
      if (error.message.includes("channel_not_found")) {
        userMessage = "I don't have access to that channel.";
      } else if (error.message.includes("not_in_channel")) {
        userMessage = "Please invite me to the channel first.";
      }
    }

    await thread.post(userMessage);
  }
});
```

## Thread Management

### Subscribing to Threads

```typescript
bot.onNewMention(async (thread, message) => {
  // Subscribe to receive follow-up messages in this thread
  await thread.subscribe();

  await thread.post("I'm listening! Send me follow-up messages in this thread.");
});

bot.onSubscribedMessage(async (thread, message) => {
  // Handle follow-up messages
  await thread.post(`Got your message: ${message.text}`);
});
```

## Typing Indicators

**Always use typing indicators** to keep Slack users informed of your agent's status.

### Basic Usage

```typescript
bot.onNewMention(async (thread, message) => {
  // Start typing indicator
  await thread.startTyping();

  // Process - typing clears automatically when you post
  const result = await processWithAI(message.text);
  await thread.post(result);
});
```

The Chat SDK handles typing indicator refresh and timeout automatically.

## Best Practices Summary

1. **Subscribe to threads** for follow-up conversations using `thread.subscribe()`
2. **Use JSX components** for rich messages instead of raw Block Kit JSON
3. **Use typing indicators** with `thread.startTyping()` for long operations
4. **Handle errors gracefully** with user-friendly messages
5. **Let Chat SDK handle ack** - no manual acknowledgment needed
6. **Use `.tsx` extension** for files with JSX components
7. **Configure tsconfig.json** with `"jsxImportSource": "chat"`
8. **Use ephemeral messages** for sensitive or temporary information
9. **Log errors** with context for debugging
10. **Use thread state** for conversational data persistence
