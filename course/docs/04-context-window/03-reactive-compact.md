---
sidebar_position: 3
title: "Reactive Compaction"
---

# Reactive Compaction

The five proactive layers described in the previous section run *before* every API call, trying to keep the context within bounds. But proactive measures can fail. Token estimation is approximate. A tool result might arrive between the budget check and the API call. The system prompt might change size. Or the context might simply be too large for any pre-emptive intervention to handle.

When this happens, the API itself becomes the arbiter. It rejects the request with an HTTP 413 error: "prompt too long." Now what?

## The Reactive Compact Flow

Reactive compaction is the system's last-resort recovery mechanism. It fires *after* the API has already rejected the request -- a fundamentally different position in the lifecycle from the proactive layers, which fire before.

The flow, visible in `query.ts` lines 1062-1175, works like this:

1. The API streams back an error response: "prompt too long" (HTTP 413)
2. The streaming loop **withholds** the error (doesn't yield it to the UI -- more on this in the [next section](./04-withholding-pattern.md))
3. After streaming ends, the code detects the withheld 413
4. First attempt: **context collapse drain** -- commit any staged collapses that haven't been applied yet
5. If that fails or doesn't free enough: **reactive compact** -- full conversation summarization
6. If compaction succeeds: retry the API call with the shorter context
7. If compaction fails: surface the withheld error to the user

Here's the decision logic after the streaming loop detects a withheld 413:

```typescript
// src/query.ts, lines 1065-1166 (simplified)
const isWithheld413 =
  lastMessage?.type === 'assistant' &&
  lastMessage.isApiErrorMessage &&
  isPromptTooLongMessage(lastMessage)

if (isWithheld413) {
  // First: drain all staged context-collapses.
  // Gated on the PREVIOUS transition not being collapse_drain_retry —
  // if we already drained and the retry still 413'd, fall through
  // to reactive compact.
  if (
    feature('CONTEXT_COLLAPSE') &&
    contextCollapse &&
    state.transition?.reason !== 'collapse_drain_retry'
  ) {
    const drained = contextCollapse.recoverFromOverflow(
      messagesForQuery,
      querySource,
    )
    if (drained.committed > 0) {
      state = {
        messages: drained.messages,
        // ...other state fields...
        transition: { reason: 'collapse_drain_retry', committed: drained.committed },
      }
      continue  // retry with collapsed context
    }
  }
}

if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,
    querySource,
    aborted: toolUseContext.abortController.signal.aborted,
    messages: messagesForQuery,
    cacheSafeParams: { systemPrompt, userContext, systemContext, toolUseContext, ... },
  })

  if (compacted) {
    const postCompactMessages = buildPostCompactMessages(compacted)
    for (const msg of postCompactMessages) {
      yield msg
    }
    state = {
      messages: postCompactMessages,
      // ...other state fields...
      hasAttemptedReactiveCompact: true,
      transition: { reason: 'reactive_compact_retry' },
    }
    continue  // retry with compacted context
  }

  // No recovery — surface the withheld error and exit
  yield lastMessage
  return { reason: 'prompt_too_long' }
}
```

Notice the layered recovery: collapse drain is tried first (cheap, preserves granular context), then reactive compact (expensive, replaces everything with a summary). This mirrors the proactive pipeline's philosophy of cheapest-first.

## Media Size Errors

There's a related but distinct recovery path for oversized images and PDFs. When the API rejects a request because an embedded image or document exceeds its size limit, the same reactive compact machinery kicks in -- but the recovery strategy is different. Instead of summarizing text, it strips the oversized media from the conversation.

```typescript
// src/query.ts, lines 1076-1084
// Media-size rejections (image/PDF/many-image) are recoverable via
// reactive compact's strip-retry. Unlike PTL, media errors skip the
// collapse drain — collapse doesn't strip images.
const isWithheldMedia =
  mediaRecoveryEnabled &&
  reactiveCompact?.isWithheldMediaSizeError(lastMessage)
```

Media errors skip the context collapse drain (collapse doesn't know how to strip images) and go directly to reactive compact's strip-and-retry logic.

## The Single-Attempt Guard

The most important safety mechanism in reactive compaction is the `hasAttemptedReactiveCompact` flag. Once reactive compact has been tried for a given turn, it won't try again -- regardless of whether it succeeded:

```typescript
// src/query.ts, lines 1152-1165
state = {
  messages: postCompactMessages,
  // ...
  hasAttemptedReactiveCompact: true,  // <-- set after first attempt
  transition: { reason: 'reactive_compact_retry' },
}
continue
```

On the retry, if the API *still* returns 413, the withheld-413 check fires again. But this time, `hasAttemptedReactiveCompact` is `true`, so `tryReactiveCompact` returns early. The code falls through to:

```typescript
// src/query.ts, lines 1168-1175
// No recovery — surface the withheld error and exit.
yield lastMessage
void executeStopFailureHooks(lastMessage, toolUseContext)
return { reason: isWithheldMedia ? 'image_error' : 'prompt_too_long' }
```

:::danger
Without the `hasAttemptedReactiveCompact` guard, a conversation that's truly too large would enter an infinite loop: compact -> still too large -> compact -> still too large... Each iteration makes an API call for the summarization, wasting tokens and time without making progress. The guard ensures exactly one attempt: compact, retry, and if it's still too big, give up and tell the user.
:::

The flag resets to `false` at natural turn boundaries (when the user sends a new message), so reactive compact is available again for the next turn. It's a per-turn circuit breaker, not a per-session one.

## The Relationship Between Proactive and Reactive

It's worth stepping back to see how these two systems interact. The proactive layers (tool result budget, snip, microcompact, collapse, auto-compact) run every turn, trying to keep the context under the threshold. Reactive compact is the backstop -- it catches what the proactive layers missed.

In a well-functioning session, reactive compact never fires. The proactive layers handle everything. Reactive compact exists for the edge cases: when token estimation was wrong, when a tool produced unexpectedly large output, when the session resumed from disk with stale size information.

The two systems are also connected by feature flags. When context collapse is enabled, it suppresses auto-compact (since collapse owns the headroom management), but reactive compact remains active as the 413 fallback. This is explicit in the source:

> "Gating here rather than in isAutoCompactEnabled() keeps reactiveCompact alive as the 413 fallback."

## Key Takeaways

- **Reactive compaction fires after the API rejects a request** (HTTP 413), not before -- it's a recovery mechanism, not a preventive one.
- **Two-stage recovery:** context collapse drain first (cheap), then full reactive compact (expensive). If both fail, the error surfaces.
- **The `hasAttemptedReactiveCompact` guard prevents infinite loops** by limiting reactive compact to one attempt per turn.
- **Media size errors** share the reactive compact infrastructure but skip context collapse (which can't strip images).
- **Proactive and reactive systems complement each other:** proactive handles the normal case, reactive catches the edge cases that slip through.
