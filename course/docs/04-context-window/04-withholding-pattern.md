---
sidebar_position: 4
title: "The Withholding Pattern"
---

# The Withholding Pattern

We've mentioned several times that errors are "withheld" from the UI during recovery. This deserves its own discussion, because it's a general UX pattern that applies far beyond context management.

The core idea: **when an error occurs that the system might be able to recover from, don't show it to the user unless recovery fails.**

This sounds obvious stated abstractly, but the implementation is subtle. In a streaming architecture where messages are yielded to the UI as they arrive, "don't show it" means actively suppressing a message that would normally flow through. Let's trace the concrete 413 scenario step by step.

## Step 1: The Error Arrives During Streaming

The streaming loop in `query.ts` processes every message from the API as it arrives. Most messages are yielded directly to the UI. But some are checked first:

```typescript
// src/query.ts, lines 799-825
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (
    contextCollapse?.isWithheldPromptTooLong(
      message,
      isPromptTooLongMessage,
      querySource,
    )
  ) {
    withheld = true
  }
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (
  mediaRecoveryEnabled &&
  reactiveCompact?.isWithheldMediaSizeError(message)
) {
  withheld = true
}
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

Four independent checks, each asking the same question: "Is this an error I might be able to recover from?" Each subsystem owns its own withholding decision. They compose via OR -- if *any* check says "withhold," the message is suppressed.

## Step 2: The Message Is Tracked, Not Discarded

The withheld message is not thrown away. It's still pushed to `assistantMessages` for internal tracking:

```typescript
// src/query.ts, lines 826-828
if (message.type === 'assistant') {
  assistantMessages.push(message)
  // ...tool_use block extraction continues normally...
}
```

This is critical. The recovery logic that runs after streaming needs to inspect the withheld message to determine what kind of error occurred. If the message were discarded entirely, the system wouldn't know what to recover from.

## Step 3: Recovery Attempts

After the streaming loop ends (line 1062+), the code inspects the last assistant message. If it was a withheld 413, recovery begins:

```typescript
// src/query.ts, lines 1070-1073
const isWithheld413 =
  lastMessage?.type === 'assistant' &&
  lastMessage.isApiErrorMessage &&
  isPromptTooLongMessage(lastMessage)
```

The recovery cascade then runs: context collapse drain, reactive compact, retry. We covered this in the [reactive compaction](./03-reactive-compact.md) section. The key point here is about *when* the user sees anything.

During all of this -- the 413, the collapse attempt, the compaction, the retry -- the user sees nothing unusual. If recovery succeeds, the conversation continues seamlessly. The error never happened, from the user's perspective.

## Step 4: Surface on Failure

Only when all recovery paths are exhausted does the withheld error get yielded to the UI:

```typescript
// src/query.ts, lines 1168-1175
// No recovery — surface the withheld error and exit. Do NOT fall
// through to stop hooks: the model never produced a valid response,
// so hooks have nothing meaningful to evaluate.
yield lastMessage
void executeStopFailureHooks(lastMessage, toolUseContext)
return { reason: isWithheldMedia ? 'image_error' : 'prompt_too_long' }
```

Now the user sees the error. But they see it with full context: all recovery has been attempted, and the situation genuinely requires their intervention.

## The Same Pattern for max_output_tokens

The withholding pattern isn't specific to 413 errors. It applies identically to `max_output_tokens` errors, where the model's response was truncated because it hit the output token limit.

The withholding check:

```typescript
// src/query.ts, lines 175-179
function isWithheldMaxOutputTokens(
  msg: Message | StreamEvent | undefined,
): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}
```

And the recovery cascade (lines 1188-1250) is a two-stage process:

1. **Escalating retry:** If the model hit the 8K default output cap, silently retry the same request with a 64K limit. No meta-message, no multi-turn dance -- just a transparent retry at higher capacity.
2. **Multi-turn continuation:** If the model hits even the escalated limit, inject a recovery message ("Output token limit hit. Resume directly -- no apology, no recap...") and continue the loop. The user sees the model keep going, not an error.

Only if the continuation count exceeds a maximum does the error surface.

The user experience difference is stark. Without withholding: the user sees an error flash, then a retry, then continuation -- confusing and alarming. With withholding: the model just... keeps going. The retry machinery is invisible.

## The Critical Invariant

:::danger
The critical invariant: **if you withhold an error, you MUST either recover from it or eventually surface it.** A withheld error that silently disappears is worse than showing the error immediately -- the user is left in a state where something failed, the system pretended nothing happened, but the underlying problem was never addressed.
:::

The codebase enforces this invariant structurally. Every withholding check has a corresponding recovery-or-surface block after the streaming loop:

- Withheld 413 → collapse drain → reactive compact → yield error
- Withheld media error → reactive compact strip → yield error  
- Withheld max_output_tokens → escalate → multi-turn → yield error

There's no path where a withheld message falls through without either being recovered from or surfaced. The comment at line 1168 makes this explicit: "No recovery -- surface the withheld error and exit."

## Why This Matters Beyond Claude Code

The withholding pattern is broadly applicable to any system with automatic recovery:

**API clients with retry logic.** If your HTTP client retries on 503, the caller shouldn't see the 503 unless retries are exhausted. Most retry libraries do this implicitly (the caller only sees the final response), but in streaming scenarios you need explicit withholding.

**Database connection pools.** When a connection drops mid-query, the pool can transparently reconnect and retry. The application layer shouldn't see the transient failure.

**Distributed systems with failover.** When a primary fails and a secondary takes over, clients should see continuous service, not an error-then-recovery sequence.

The common thread: **internal retry machinery should be invisible to the layer above.** The user (or the calling code) should see either success or a final, unrecoverable failure -- never the intermediate states of a recovery process.

The implementation challenge is always the same: you need to *hold* the error message while attempting recovery, and you need to guarantee that held messages eventually resolve (either recovered from or surfaced). Claude Code's approach -- a `withheld` boolean that gates `yield`, combined with explicit recovery-or-surface blocks for every error type -- is one clean way to achieve this.

## Key Takeaways

- **The withholding pattern suppresses recoverable errors from the UI** until recovery either succeeds (error disappears) or fails (error surfaces).
- **Withheld messages are tracked, not discarded** -- recovery logic needs them to determine what went wrong.
- **Four independent subsystems** (context collapse, reactive compact, media recovery, max_output_tokens) each make their own withholding decisions, composed via OR.
- **The critical invariant:** every withheld error must be either recovered from or eventually surfaced. Silent swallowing is a bug worse than showing the error.
- **The pattern generalizes** to any system with transparent retry or failover: internal recovery should be invisible to the layer above.
