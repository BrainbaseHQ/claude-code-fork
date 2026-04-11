---
sidebar_position: 2
title: "Three Execution Models"
---

# Three Execution Models

Once you decide to spawn a subagent, the next design question is: *how does it relate to its parent?* This is not a theoretical distinction. The execution model determines whether the parent blocks or continues, whether the child can ask for user permission, and what happens when the user presses Ctrl+C.

Claude Code implements three execution models, each with different boundaries for abort control, state sharing, and permission prompts. Understanding these boundaries is essential for reasoning about subagent behavior in production.

## Model 1: Synchronous Subagent

**`run_in_background: false` (the default)**

A synchronous subagent is the simplest model. The parent spawns it and blocks, waiting for the result before continuing. Think of it as a function call -- the parent's query loop is paused while the child runs.

The key characteristic: the child shares the parent's infrastructure.

```typescript
// src/tools/AgentTool/runAgent.ts, line 520
// Determine abortController:
// - Override takes precedence
// - Async agents get a new unlinked controller (runs independently)
// - Sync agents share parent's controller
const agentAbortController = override?.abortController
  ? override.abortController
  : isAsync
    ? new AbortController()
    : toolUseContext.abortController
```

When the user presses Ctrl+C, the parent's `AbortController` fires -- and because the child shares it, both are cancelled. This is the right behavior: the user expects cancellation to cancel everything visible on screen.

State changes propagate too. In `createSubagentContext`, the `shareSetAppState` flag controls whether the child's `setAppState` calls actually reach the root store:

```typescript
// src/utils/forkedAgent.ts, line 410
setAppState: overrides?.shareSetAppState
  ? parentContext.setAppState
  : () => {},
```

For sync agents, `shareSetAppState` is `true` (set in `runAgent.ts` line 709: `shareSetAppState: !isAsync`). State updates from the child -- like permission grants or tool configuration changes -- propagate to the parent immediately.

And because the child is running in the foreground with the user watching, it can show permission prompts:

```typescript
// src/tools/AgentTool/runAgent.ts, line 440
const shouldAvoidPrompts =
  canShowPermissionPrompts !== undefined
    ? !canShowPermissionPrompts
    : agentPermissionMode === 'bubble'
      ? false
      : isAsync
```

For sync agents (`isAsync` is `false`), `shouldAvoidPrompts` evaluates to `false` -- the child can show permission dialogs to the user.

**Use case**: Focused research tasks where the parent needs the result before continuing. "Read these 5 files and tell me the API surface" -- the parent can't plan the refactoring until it knows the answer.

## Model 2: Asynchronous (Background) Subagent

**`run_in_background: true`**

An asynchronous subagent runs independently. The parent spawns it and continues immediately -- it doesn't wait for the result. The child runs in the background, and the parent is notified when it completes.

The boundaries are stricter here because the child operates without the user's attention:

**Own AbortController**: The child gets a new, unlinked `AbortController`. When the user presses Ctrl+C on the parent, the child keeps running. Background agents are killed explicitly via separate mechanisms (like `chat:killAgents`), not by parent cancellation.

**Isolated state**: `setAppState` is a no-op (`shareSetAppState: false`). The child can't accidentally modify the parent's state -- no surprise permission changes, no unexpected UI updates. However, task registration (bash processes, MCP tasks) still reaches the root store via `setAppStateForTasks`, ensuring background processes are tracked and can be killed:

```typescript
// src/utils/forkedAgent.ts, line 414
// Task registration/kill must always reach the root store, even when
// setAppState is a no-op — otherwise async agents' background bash tasks
// are never registered and never killed (PPID=1 zombie).
setAppStateForTasks:
  parentContext.setAppStateForTasks ?? parentContext.setAppState,
```

**No permission prompts**: `shouldAvoidPermissionPrompts` is set to `true`. The child auto-denies any permission request it can't resolve automatically. This prevents background agents from silently blocking on a prompt nobody sees.

**Use case**: Long-running tasks and parallel work. "Run the full test suite," "Refactor module A while I work on module B," or any task where the parent doesn't need the result immediately.

## Model 3: Bubble Mode

**`permissionMode: 'bubble'`**

Bubble mode is a hybrid. The child runs asynchronously like a background agent, but permission prompts *surface in the parent's terminal* rather than being auto-denied.

The permission logic in `agentGetAppState` handles this case explicitly:

```typescript
// src/tools/AgentTool/runAgent.ts, line 440
const shouldAvoidPrompts =
  canShowPermissionPrompts !== undefined
    ? !canShowPermissionPrompts
    : agentPermissionMode === 'bubble'
      ? false  // bubble mode: always show prompts
      : isAsync
```

When `agentPermissionMode` is `'bubble'`, `shouldAvoidPrompts` is `false` regardless of whether the agent is async. The child can ask for permission -- but it does so through the parent's UI, not its own.

There's a subtlety here. Because bubble agents are async but *can* show prompts, the system enables `awaitAutomatedChecksBeforeDialog` -- it waits for automated permission checks (classifiers, hooks) to resolve before interrupting the user:

```typescript
// src/tools/AgentTool/runAgent.ts, line 458
// For background agents that can show prompts, await automated checks
// (classifier, permission hooks) before showing the permission dialog.
if (isAsync && !shouldAvoidPrompts) {
  toolPermissionContext = {
    ...toolPermissionContext,
    awaitAutomatedChecksBeforeDialog: true,
  }
}
```

This keeps the user experience clean: the user is only interrupted when automated systems can't handle the permission request.

**Use case**: Background agents that need write access but should ask before destructive operations. The fork subagent path uses bubble mode by default -- fork children run independently but can surface permission requests to the user when they need to write files or run commands.

## Comparison

| Aspect | Sync | Async (Background) | Bubble |
|-|-|-|-|
| AbortController | Shared with parent | Own (independent) | Own (independent) |
| State sharing | Full (`setAppState` propagates) | Isolated (no-op `setAppState`) | Isolated (no-op `setAppState`) |
| Permission prompts | Shown to user directly | Auto-denied | Surfaced in parent terminal |
| Parent blocks? | Yes | No | No |
| `isAsync` | `false` | `true` | `true` |
| Primary use case | Focused research, sequential work | Parallel work, long tasks | Background work needing write access |

## How Async/Sync Is Determined

The decision isn't purely based on `run_in_background`. Several conditions can force async mode:

```typescript
// src/tools/AgentTool/AgentTool.tsx, line 567
const shouldRunAsync = (
  run_in_background === true ||
  selectedAgent.background === true ||
  isCoordinator ||
  forceAsync ||
  assistantForceAsync ||
  (proactiveModule?.isProactiveActive() ?? false)
) && !isBackgroundTasksDisabled;
```

An agent runs async if the model requested it (`run_in_background`), the agent definition requires it (`background: true`), the system is in coordinator mode, the fork experiment is active (all spawns are forced async), or proactive mode is active. The `isBackgroundTasksDisabled` flag provides an escape hatch for environments that can't support background execution.

:::tip
The choice between sync and async comes down to a simple question: does the parent need the result to continue? If you're gathering information to answer a question, use sync -- the parent can't proceed without the answer. If you're kicking off parallel work or a long-running task, use async -- the parent has other things to do while the child works.
:::

## Key Takeaways

- **Sync subagents share everything with the parent**: abort controller, state, permission UI. They're function calls -- the parent blocks until they return.
- **Async subagents are isolated**: own abort controller, no-op state writes, auto-denied permissions. They're fire-and-forget tasks that notify on completion.
- **Bubble mode is the hybrid**: async execution with permission prompts that surface in the parent's terminal. Used when background work needs user authorization for destructive operations.
- **The boundaries are enforced structurally**, not by convention. A no-op `setAppState` can't modify state. A `shouldAvoidPermissionPrompts` flag auto-denies prompts. These are hard boundaries that hold regardless of what the model tries to do.
- **Task registration always reaches the root store** via `setAppStateForTasks`, even for isolated agents. This prevents zombie processes -- background bash tasks spawned by async agents must be tracked so they can be killed on cleanup.
