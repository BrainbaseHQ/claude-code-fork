---
sidebar_position: 3
title: "Streaming API Responses"
---

# Streaming API Responses

An API response from Claude doesn't arrive as a single payload. It streams -- chunks of text, thinking blocks, and tool calls arrive over 5 to 30 seconds. The query loop must process these chunks incrementally: rendering text to the UI as it arrives, detecting tool calls as they complete, and handling errors that surface mid-stream.

This section examines the streaming inner loop -- the `for await` block at the heart of each query iteration.

## The Streaming Loop

The streaming loop sits inside a `while (attemptWithFallback)` wrapper (for model fallback retry), but its core shape is a `for await` over the model's streaming output:

```typescript
// src/query.ts, line 659-863 (simplified)
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fallbackModel,
    maxOutputTokensOverride,
    querySource,
    // ... many more options
  },
})) {
  // Process each streamed message
  yield message          // forward to UI
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    // extract tool_use blocks, feed to streaming executor
  }
}
```

The `deps.callModel` is `queryModelWithStreaming` from the API layer (see `src/query/deps.ts`). It yields `Message` and `StreamEvent` objects as they arrive from the API. The query loop's job is to forward them, collect them, and react to what's inside them.

## What Happens Inside the Loop

For each streamed message, the loop performs several operations:

**1. Streaming fallback detection (line 712-740):** If a streaming fallback occurred (the primary model became unavailable mid-stream), all assistant messages collected so far are orphaned. The loop yields tombstones for them and clears all accumulated state:

```typescript
// src/query.ts, line 712-740
if (streamingFallbackOccured) {
  // Yield tombstones for orphaned messages so they're removed
  // from UI and transcript. These partial messages (especially
  // thinking blocks) have invalid signatures that would cause
  // "thinking blocks cannot be modified" API errors.
  for (const msg of assistantMessages) {
    yield { type: 'tombstone' as const, message: msg }
  }
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0
  needsFollowUp = false

  if (streamingToolExecutor) {
    streamingToolExecutor.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)
  }
}
```

**2. Backfilling tool inputs (line 742-786):** Before yielding assistant messages, the loop clones and augments `tool_use` blocks with derived fields for SDK consumers and transcript serialization. The original message is left untouched -- mutating it would break prompt caching because the API matches cached prompts by exact byte content.

```typescript
// src/query.ts, line 747-786
let yieldMessage: typeof message = message
if (message.type === 'assistant') {
  let clonedContent: typeof message.message.content | undefined
  for (let i = 0; i < message.message.content.length; i++) {
    const block = message.message.content[i]!
    if (block.type === 'tool_use' && typeof block.input === 'object') {
      const tool = findToolByName(toolUseContext.options.tools, block.name)
      if (tool?.backfillObservableInput) {
        const inputCopy = { ...block.input }
        tool.backfillObservableInput(inputCopy)
        // Only clone when backfill ADDED fields, not overwrote
        const addedFields = Object.keys(inputCopy).some(
          k => !(k in (block.input as Record<string, unknown>)),
        )
        if (addedFields) {
          clonedContent ??= [...message.message.content]
          clonedContent[i] = { ...block, input: inputCopy }
        }
      }
    }
  }
  if (clonedContent) {
    yieldMessage = {
      ...message,
      message: { ...message.message, content: clonedContent },
    }
  }
}
```

This is a subtle but important pattern: the message yielded to the UI and transcript can differ from the message stored for the API. The API copy stays byte-identical for cache stability; the UI copy carries additional derived fields.

**3. Error withholding (line 788-825):** Certain error messages are withheld from the UI until the loop knows whether recovery will succeed. The loop pushes them to `assistantMessages` (so recovery checks find them) but does not yield them:

```typescript
// src/query.ts, line 799-825
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, ...)) {
    withheld = true
  }
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) {
  withheld = true
}
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

Three categories of errors are withheld:

- **Prompt too long (413)** -- reactive compact or context collapse might recover
- **Media size errors** -- image/PDF stripping might recover
- **Max output tokens** -- escalation or multi-turn recovery might continue the response

If recovery fails, the withheld error is yielded later (see the [decision tree](./04-continue-decision-tree.md)). If recovery succeeds, the error is never shown to the user -- they see only the successful retry.

**4. Tool use detection and streaming execution (line 826-862):** As tool_use blocks arrive, they're pushed to `toolUseBlocks[]` and, if streaming tool execution is enabled, immediately queued in the `StreamingToolExecutor`. Completed tool results are yielded from the streaming loop itself -- tools can finish while the model is still generating text:

```typescript
// src/query.ts, line 826-862
if (message.type === 'assistant') {
  assistantMessages.push(message)

  const msgToolUseBlocks = message.message.content.filter(
    content => content.type === 'tool_use',
  ) as ToolUseBlock[]
  if (msgToolUseBlocks.length > 0) {
    toolUseBlocks.push(...msgToolUseBlocks)
    needsFollowUp = true
  }

  if (streamingToolExecutor && !aborted) {
    for (const toolBlock of msgToolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, message)
    }
  }
}

if (streamingToolExecutor && !aborted) {
  for (const result of streamingToolExecutor.getCompletedResults()) {
    if (result.message) {
      yield result.message
      toolResults.push(...)
    }
  }
}
```

## The Tombstone Pattern

When a streaming fallback occurs (the primary model goes down mid-response and a fallback model takes over), the partial response from the first model is useless -- and worse, dangerous. Thinking blocks carry model-specific cryptographic signatures, and replaying a primary model's thinking block to a fallback model causes a hard API error.

The solution: yield **tombstone** messages. A tombstone tells the UI "this message existed but is now dead -- remove it from your display and transcript." The UI treats tombstones as deletion markers, and the transcript serializer skips them.

```mermaid
sequenceDiagram
    participant API as Claude API
    participant Loop as Query Loop
    participant UI as REPL / SDK

    API->>Loop: assistant message (partial, model A)
    Loop->>UI: yield assistant message
    Note over API: Model A goes down
    API->>Loop: FallbackTriggeredError (or streaming callback)
    Loop->>UI: yield tombstone (orphaned message)
    Note over UI: Remove orphaned message from display
    API->>Loop: assistant message (complete, model B)
    Loop->>UI: yield assistant message
    Loop->>UI: yield tool results (if any)
```

## The Withholding Pattern

The withholding pattern solves a coordination problem. When the API returns a 413 (prompt too long), the model hasn't produced a useful response -- only an error. But the query loop has recovery paths (context collapse, reactive compact) that might fix the problem and retry. If the loop yields the error immediately, SDK consumers see it and terminate the session. The recovery loop keeps running, but nobody is listening.

The fix: withhold recoverable errors from `yield` while still pushing them to `assistantMessages`. After the streaming loop ends, the decision tree checks for withheld errors and either recovers (the error is never yielded) or gives up (the error is yielded at that point). This is the same pattern used for `max_output_tokens` errors -- the loop might recover by injecting a "resume" message, and the user should only see the error if recovery exhausts.

## The Backfilling Pattern

Tool inputs sometimes need additional fields for observability. For example, a file tool might receive a relative path from the model, but the UI and SDK want to see the resolved absolute path. The `backfillObservableInput` hook on each tool definition can add these fields.

But there's a cache-stability constraint: the API matches cached prompts by byte content. If the loop mutates the original tool_use block to add a `resolved_path` field, the next API call will include that field, the bytes won't match the cached version, and the prompt cache misses -- adding seconds of latency and dollars of cost.

The solution: clone the message content only when backfill adds new fields (not when it overwrites existing ones), yield the clone to the UI, and push the original to `assistantMessages` for the API. Two views of the same message, each optimized for its consumer.

## Key Takeaways

- **The streaming loop processes messages incrementally** as they arrive from the API, yielding each to the UI in real time while collecting them for the loop's post-streaming logic.
- **Tombstones handle orphaned messages** when model fallbacks occur mid-stream, preventing invalid thinking signatures from corrupting the conversation.
- **Error withholding decouples error visibility from error handling** -- recoverable errors are hidden from consumers until recovery either succeeds (error disappears) or fails (error surfaces).
- **Backfilling creates two views of each message**: a cache-stable original for the API and an augmented clone for the UI/SDK, avoiding prompt cache invalidation while still providing rich observability.
- **Streaming tool execution starts during the stream**, not after -- tool_use blocks are handed to the `StreamingToolExecutor` as they arrive, overlapping tool I/O with the remainder of the model's response.
