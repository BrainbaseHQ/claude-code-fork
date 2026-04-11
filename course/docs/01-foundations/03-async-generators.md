---
sidebar_position: 3
title: "AsyncGenerators: Streaming by Design"
---

# AsyncGenerators: Streaming by Design

The query loop at the heart of Claude Code is an async generator. This isn't an incidental implementation choice — it's a deliberate architectural decision that shapes how the entire system handles streaming, cancellation, and composition. Before we look at the actual code, let's build up to async generators from first principles.

## From Functions to Generators

A regular function takes input and produces one output:

```typescript
function square(n: number): number {
  return n * n
}

const result = square(4)  // 16
```

A **generator** takes input and produces *many* outputs, one at a time:

```typescript
function* range(start: number, end: number): Generator<number> {
  for (let i = start; i < end; i++) {
    yield i
  }
}

for (const n of range(0, 5)) {
  console.log(n)  // 0, 1, 2, 3, 4
}
```

The `yield` keyword is the key difference. Where `return` exits the function and delivers a final value, `yield` *pauses* the function and delivers an intermediate value. The function resumes from where it left off the next time the caller asks for the next value.

An **async generator** combines this with `async/await` — it can yield values over time, with asynchronous operations between yields:

```typescript
async function* readLines(path: string): AsyncGenerator<string> {
  const file = await open(path)
  for await (const line of file.readLines()) {
    yield line
  }
}

for await (const line of readLines('/tmp/data.txt')) {
  console.log(line)
}
```

Each `yield` delivers a line to the caller. The caller processes it at its own pace. If the caller stops iterating (breaks out of the loop), the generator cleans up. This is fundamentally different from callbacks (where the producer controls the pace) or promises (where you get one value).

## A Simplified Query Loop

Now imagine a simplified version of what Claude Code's query loop does. The model produces a response that might contain text to show the user *and* tool calls to execute. We need to stream all of this incrementally:

```typescript
async function* simpleQueryLoop(
  messages: Message[]
): AsyncGenerator<StreamEvent | Message> {
  while (true) {
    // Stream the model's response
    for await (const event of callModelStreaming(messages)) {
      yield event  // UI sees tokens as they arrive
    }

    const toolCalls = extractToolCalls(lastMessage(messages))
    if (toolCalls.length === 0) break  // Model is done

    // Execute tools and yield results
    for (const call of toolCalls) {
      const result = await executeTool(call)
      const message = createToolResultMessage(result)
      messages.push(message)
      yield message  // UI sees each tool result
    }
  }
}
```

The caller — whether it's the REPL, the SDK, or a test — consumes this with a `for await` loop:

```typescript
for await (const event of simpleQueryLoop(messages)) {
  if (event.type === 'stream') renderToken(event.text)
  if (event.type === 'user')   renderToolResult(event)
}
```

The generator doesn't need to know what the caller does with each event. It doesn't need callback registrations, event emitter patterns, or observable subscriptions. It just yields values and the caller pulls them.

## yield vs return

Generators have two ways to produce values, and the distinction matters:

- **`yield`** sends an intermediate value. The generator pauses and the caller receives the value in the `for-of` loop. The generator resumes when the caller asks for the next value.
- **`return`** sends a final value and terminates the generator. The caller does *not* see this value in a `for-of` loop — it's available only through the generator's `.next()` protocol.

In Claude Code, this distinction is used architecturally. The `query()` function yields streaming events (what the UI needs to render) and *returns* a `Terminal` value (why the loop stopped):

```typescript
// src/query.ts — lines 219-239
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

The `AsyncGenerator` generic takes two type parameters here: the **yield type** (the union of things streamed to the caller — `StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage`) and the **return type** (`Terminal` — the reason the query loop terminated, such as the model stopping naturally, hitting a turn limit, or encountering an unrecoverable error).

This separation is clean. The UI layer consumes the yielded events in real-time. The orchestration layer (REPL or QueryEngine) examines the return value to decide what to do next — maybe prompt the user for more input, maybe run post-turn hooks, maybe terminate the session.

## yield*: Generator Delegation

Notice the `yield*` on this line:

```typescript
const terminal = yield* queryLoop(params, consumedCommandUuids)
```

`yield*` is *delegation*. It means "run this other generator, and everything it yields, yield from me too. When it returns, give me its return value." The caller can't tell whether a yielded value came from `query()` or from the inner `queryLoop()` — they all flow through the same stream.

This is how Claude Code composes generators. `query()` delegates to `queryLoop()`, which in turn may delegate to tool execution generators. Each level can yield its own events, and they all bubble up to the caller through the `yield*` chain. It's generators all the way down.

The comment in the source code highlights an important subtlety:

```typescript
// Only reached if queryLoop returned normally. Skipped on throw (error
// propagates through yield*) and on .return() (Return completion closes
// both generators).
```

When an error is thrown inside `queryLoop()`, it propagates through `yield*` and out of `query()` — the caller sees the exception. When the caller calls `.return()` on the `query()` generator (for cancellation), both generators close. The `yield*` mechanism handles all of this automatically.

## Why Generators (and Not Something Else)

There are several patterns for streaming asynchronous data. Why async generators specifically?

**Backpressure.** The consumer controls the pace. If the UI is slow to render, the generator naturally pauses at the next `yield` until the consumer pulls again. With callbacks or event emitters, the producer runs at its own pace and the consumer must buffer or drop events. In a system that streams tokens from an API while simultaneously executing tools, backpressure prevents memory bloat.

**Cancellation.** Calling `generator.return()` triggers cleanup. Any `try/finally` blocks in the generator run, resources are released, and the generator is marked as done. This maps perfectly to user interruptions — when the user hits Escape, the REPL calls `.return()` on the query generator, which cascades through `yield*` to close the entire chain of nested generators. No separate cancellation token system needed (though Claude Code also uses `AbortController` for imperative cancellation of non-generator code like fetch requests).

**Composition via yield\*.** Generators delegate to sub-generators with `yield*`, creating a natural call-stack-like structure where each level can add its own pre/post processing. `query()` wraps `queryLoop()` with command lifecycle notifications. `queryLoop()` wraps individual tool executions. This composability would require significant ceremony with callbacks or observables.

**Testability.** A generator is a plain object with a `.next()` method. You can step through it one yield at a time, inspect each yielded value, and assert on the sequence of events. No need to mock event emitters, subscribe to observables, or manage callback chains.

:::tip
If you've used Python's `yield` or C#'s `IAsyncEnumerable`, the concept is the same. TypeScript's async generators combine the streaming semantics of iterators with the async/await model. The `for await...of` loop is the consumer-side counterpart, and `yield*` is the composition primitive that makes it all stack.
:::

## Key Takeaways

- **Async generators are the streaming primitive.** The `query()` function — the core of the agent loop — is an async generator. Everything it produces (streaming tokens, tool results, progress updates) is yielded to the caller incrementally.

- **yield is for the stream, return is for the outcome.** Yielded values flow to the UI in real-time. The return value (`Terminal`) tells the orchestration layer why the loop stopped. This separates the "what happened along the way" from the "what was the final result."

- **yield\* composes generators transparently.** `query()` delegates to `queryLoop()` via `yield*`, which means every event from the inner generator flows through the outer one. The caller sees a single unified stream. This pattern repeats throughout the codebase wherever streaming operations are composed.

- **Backpressure and cancellation come free.** The pull-based nature of generators means the consumer controls the pace (backpressure), and `generator.return()` provides a clean cancellation mechanism that cascades through the entire `yield*` chain. These properties are difficult to achieve with push-based alternatives like callbacks or event emitters.

- **The pattern is language-agnostic.** Whether you implement your harness in TypeScript, Python, Rust, or Go, the concept of a pull-based streaming loop with delegation and cancellation applies. The syntax varies; the architecture transfers.
