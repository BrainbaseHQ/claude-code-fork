---
sidebar_position: 3
title: "Hooks System and Memory Management"
---

# Hooks System and Memory Management

Two concerns that seem unrelated but share a deep structural pattern: extensibility and resource management. Every extension point in an agent harness -- hooks, subagents, MCP servers -- is also a potential leak point. Understanding both sides is essential for production systems.

## Part 1: The Hooks System

Hooks are the primary extension mechanism in Claude Code. They are commands (shell scripts, prompts, agent invocations, HTTP endpoints) that execute in response to lifecycle events. They let users and organizations customize behavior without modifying the harness code itself.

### Hook Events

The harness defines 25 hook events, covering every significant moment in the agent lifecycle:

```typescript
// src/entrypoints/sdk/coreTypes.ts (lines 25-53)
export const HOOK_EVENTS = [
  'PreToolUse',
  'PostToolUse',
  'PostToolUseFailure',
  'Notification',
  'UserPromptSubmit',
  'SessionStart',
  'SessionEnd',
  'Stop',
  'StopFailure',
  'SubagentStart',
  'SubagentStop',
  'PreCompact',
  'PostCompact',
  'PermissionRequest',
  'PermissionDenied',
  'Setup',
  'TeammateIdle',
  'TaskCreated',
  'TaskCompleted',
  'Elicitation',
  'ElicitationResult',
  'ConfigChange',
  'WorktreeCreate',
  'WorktreeRemove',
  'InstructionsLoaded',
  'CwdChanged',
  'FileChanged',
] as const
```

The events fall into natural categories. **Tool lifecycle** (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`) -- run checks before a tool executes, react to results after. **Session lifecycle** (`SessionStart`, `SessionEnd`, `Stop`, `StopFailure`) -- set up context on start, clean up on end. **Subagent lifecycle** (`SubagentStart`, `SubagentStop`) -- track when agents are spawned and completed. **Context management** (`PreCompact`, `PostCompact`, `InstructionsLoaded`) -- respond to compaction and instruction loading. **Permission events** (`PermissionRequest`, `PermissionDenied`) -- audit or modify permission decisions.

### Registration: Three Sources

Hooks can come from three places, each with different scope and lifetime:

1. **Settings files** (`settings.json`) -- User-configured hooks that apply globally or per-project. These are loaded at startup via `captureHooksConfigSnapshot` and persist across sessions.

2. **Agent frontmatter** -- Hooks defined in an agent's markdown frontmatter, scoped to that agent's lifetime. The `registerFrontmatterHooks` function handles this:

```typescript
// src/utils/hooks/registerFrontmatterHooks.ts (lines 18-23)
export function registerFrontmatterHooks(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  sessionId: string,
  hooks: HooksSettings,
  sourceName: string,
  isAgent: boolean = false,
): void {
```

A notable detail: when `isAgent` is true, `Stop` hooks are automatically converted to `SubagentStop`. This is because subagents trigger `SubagentStop`, not `Stop`, when they complete -- an agent defining hooks for its own completion must use the right event.

3. **Session hooks** -- In-memory hooks registered programmatically during a session. These include both command hooks (shell scripts) and function hooks (TypeScript callbacks). Function hooks are used internally for structured output enforcement and validation:

```typescript
// src/utils/hooks/sessionHooks.ts (lines 93-115)
export function addFunctionHook(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  sessionId: string,
  event: HookEvent,
  matcher: string,
  callback: FunctionHookCallback,
  errorMessage: string,
  options?: { timeout?: number; id?: string },
): string {
  const id = options?.id || `function-hook-${Date.now()}-${Math.random()}`
  const hook: FunctionHook = {
    type: 'function',
    id,
    timeout: options?.timeout || 5000,
    callback,
    errorMessage,
  }
  addHookToSession(setAppState, sessionId, event, matcher, hook)
  return id
}
```

### The Execution Pipeline

When a hook event fires, execution follows a clear pipeline:

1. **Fast bail-out.** `hasHookForEvent()` checks whether any hooks are configured for the event at all, across settings, registered hooks, and session hooks. If none exist, skip the entire pipeline -- no input construction, no matching, no execution.

2. **Matching.** `getMatchingHooks()` finds hooks whose matchers match the event's context. For tool hooks, the matcher is the tool name. For session events, it might be the trigger source. Each hook event type has a specific match query:

```typescript
// src/utils/hooks.ts (lines 1616-1668)
switch (hookInput.hook_event_name) {
  case 'PreToolUse':
  case 'PostToolUse':
  case 'PostToolUseFailure':
  case 'PermissionRequest':
  case 'PermissionDenied':
    matchQuery = hookInput.tool_name
    break
  case 'SessionStart':
    matchQuery = hookInput.source
    break
  case 'Notification':
    matchQuery = hookInput.notification_type
    break
  // ... (each event type has its own matching field)
}
```

3. **Execution.** The `executeHooks()` generator runs matching hooks in sequence, yielding results as they complete. Each hook receives structured JSON input (the `HookInput` for its event type) via stdin, and its output is parsed as structured JSON.

4. **Result processing.** Hook output is validated against a Zod schema. The output can influence the system in several ways: allow or deny a tool invocation, inject additional context into the conversation, or block the agent from stopping.

:::info
Hooks are the extension point for the harness. They are how organizations enforce policies (deny certain Bash commands), add context (inject deployment status before every query), or integrate with external systems (log tool usage to an audit trail). They are also how agents define their own lifecycle behavior through frontmatter.
:::

### Performance: The Internal Hook Fast Path

Not all hooks are user-defined shell commands. The system also uses internal callback hooks for file access tracking, attribution, and other bookkeeping. These are lightweight TypeScript callbacks that return `{}` and don't need abort signals or progress tracking.

The `executeHooks` function detects when all matching hooks are internal and takes a fast path that skips span creation, progress tracking, and JSON output parsing. The measured improvement: 6.01us down to ~1.8us per `PostToolUse` hit -- a 70% reduction that matters when it runs on every tool invocation in a session.

## Part 2: Memory Management for Long Sessions

Agent sessions can run for hours. Without careful memory management, the process grows until it crashes, slows, or runs out of heap space. The dangerous leaks are not the obvious ones (forgotten `setInterval`, unclosed file handles). They are the subtle ones: a closure that captures a large array, an event listener that is never removed, a cache that grows unbounded.

### File State Cache with Size Limits

The `FileStateCache` (`src/utils/fileStateCache.ts`) wraps an LRU cache with both entry count and byte size limits:

```typescript
// src/utils/fileStateCache.ts (lines 30-39)
export class FileStateCache {
  private cache: LRUCache<string, FileState>

  constructor(maxEntries: number, maxSizeBytes: number) {
    this.cache = new LRUCache<string, FileState>({
      max: maxEntries,
      maxSize: maxSizeBytes,
      sizeCalculation: value => Math.max(1, Buffer.byteLength(value.content)),
    })
  }
}
```

The default limits: 100 entries, 25MB total. The `sizeCalculation` function ensures each entry's size is measured by its content's byte length, not just its key count. The `Math.max(1, ...)` guard handles empty content without breaking the LRU's size accounting.

This cache stores the content of files the agent has read, used for detecting changes and providing context to the compaction system. Without size limits, a session that reads many large files (node_modules, compiled output, large data files) would accumulate unbounded memory.

### Closure Memory Retention

A pattern in `src/query.ts` illustrates how closure capture can cause insidious memory growth:

```typescript
// src/query.ts (lines 582-590)
// Create fetch wrapper once per query session to avoid memory retention.
// Each call to createDumpPromptsFetch creates a closure that captures
// the request body. Creating it once means only the latest request body
// is retained (~700KB), instead of all request bodies from the session
// (~500MB for long sessions).
const dumpPromptsFetch = config.gates.isAnt
  ? createDumpPromptsFetch(toolUseContext.agentId ?? config.sessionId)
  : undefined
```

The `createDumpPromptsFetch` function creates a closure that captures the last API request body for debugging. If you created a new closure per API call (inside the loop), each closure would retain its own copy of the request body. Over a long session with hundreds of API calls, that is hundreds of ~700KB request bodies kept alive by closures -- 500MB or more of dead data. Creating the closure once, outside the loop, means only the latest body is retained.

### WeakRef for Abort Listeners

We covered this in the abort controller section, but it bears repeating in the memory context. The `createChildAbortController` function uses `WeakRef` so that a parent controller does not retain a strong reference to abandoned child controllers. This prevents a common leak pattern: long-lived parent objects keeping short-lived children alive through event listeners.

### Explicit Cleanup in Finally Blocks

When a subagent completes, its `finally` block performs explicit cleanup:

```typescript
// src/tools/AgentTool/runAgent.ts (lines 816-830)
} finally {
  // Clean up agent-specific MCP servers
  await mcpCleanup()
  // Clean up agent's session hooks
  if (agentDefinition.hooks) {
    clearSessionHooks(rootSetAppState, agentId)
  }
  // Clean up prompt cache tracking state for this agent
  if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
    cleanupAgentTracking(agentId)
  }
  // Release cloned file state cache memory
  agentToolUseContext.readFileState.clear()
  // Release the cloned fork context messages
  initialMessages.length = 0
}
```

The `initialMessages.length = 0` line is worth noting. This is a deliberate GC hint. When a subagent is spawned with forked context, `initialMessages` holds a copy of the parent's conversation history. Setting the array's length to 0 releases those message objects. Without this, the array would be kept alive by the generator's closure until the parent's scope is fully collected -- which, in a deeply nested subagent tree, could be much later.

The same pattern -- `array.length = 0` -- appears in the query loop for `assistantMessages`, `toolResults`, and `toolUseBlocks` during streaming fallback retries, preventing stale data from accumulating across retry attempts.

### Session Hook State: Map vs. Record

A design choice in the session hooks system (`src/utils/hooks/sessionHooks.ts`) shows how data structure selection affects memory under concurrency:

```typescript
// src/utils/hooks/sessionHooks.ts (lines 49-61)
// Map (not Record) so .set/.delete don't change the container's identity.
// Mutator functions mutate the Map and return prev unchanged, letting
// store.ts's Object.is(next, prev) check short-circuit and skip listener
// notification.
//
// This matters under high-concurrency workflows: parallel() with N
// schema-mode agents fires N addFunctionHook calls in one synchronous
// tick. With a Record + spread, each call cost O(N) to copy the growing
// map (O(N^2) total) plus fired all ~30 store listeners. With Map: .set()
// is O(1), return prev means zero listener fires.
export type SessionHooksState = Map<string, SessionStore>
```

Using a `Map` instead of a plain object means hook registration is O(1) instead of O(N) (no spread-copy), and returning the same reference (`prev`) skips store listener notifications entirely. Under a parallel workflow with 20 agents each registering hooks in the same tick, this is the difference between 20 O(N) copies plus 600 listener fires versus 20 O(1) mutations and zero fires.

:::danger
In long-running agent processes, the most dangerous leaks are subtle: a closure that captures a large array, an event listener that is never removed, a cache that grows unbounded. Each is small on its own, but over hundreds of tool invocations they compound. A 700KB request body retained per API call becomes 500MB over a session. A listener per tool invocation becomes thousands of dead handlers on a single signal.
:::

## Key Takeaways

- **Hooks are the agent harness's extension point.** They execute shell commands, prompts, agent invocations, or HTTP calls in response to 25 lifecycle events. Registration comes from settings files, agent frontmatter, or programmatic session hooks.
- **The hook execution pipeline is optimized for the common case.** Fast bail-out when no hooks are configured, a fast path for internal callbacks, and structured matching based on event-specific fields.
- **Memory management in long sessions requires explicit patterns**: LRU caches with size limits, WeakRef for parent-child listener relationships, closure-scope management to avoid retaining stale data, and explicit cleanup in `finally` blocks.
- **Every extension point is a potential leak point.** The pattern is always the same: register on entry, execute during, clean up in `finally`. Hooks clean up via `clearSessionHooks`. Subagents clean up file state caches, MCP connections, and tracking state. The `finally` block is not optional -- it is where correctness lives.
