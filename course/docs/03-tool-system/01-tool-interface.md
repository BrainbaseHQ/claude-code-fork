---
sidebar_position: 1
title: "The Tool Interface"
---

# The Tool Interface

Every capability that a production agent exposes to the language model -- reading files, running shell commands, searching codebases, spawning subagents -- is expressed as a **tool**. In Claude Code, the `Tool` type is the contract that all these capabilities share. Understanding this contract is the first step to understanding the entire tool system.

## The Shape of a Tool

The `Tool` type lives in `src/Tool.ts` and contains roughly 30+ methods and properties. That might sound intimidating, but most of them exist for rendering, analytics, or edge-case handling. The core contract is small. Here are the methods that matter most, grouped by purpose:

### Identity and Schema

```typescript
// src/Tool.ts (lines 362-400)
export type Tool<Input, Output, P> = {
  readonly name: string
  readonly inputSchema: Input          // Zod schema for validating model-provided input
  outputSchema?: z.ZodType<unknown>    // Optional schema for the tool's output
  aliases?: string[]                   // Backwards-compatible names after renames
  // ...
}
```

Every tool has a `name` that the model uses to invoke it, and an `inputSchema` (a Zod schema) that validates the arguments the model provides. The `outputSchema` is optional -- not all tools declare one yet, but when present it gives downstream code a typed contract for the result.

### Execution

```typescript
// src/Tool.ts (lines 379-385)
call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<P>,
): Promise<ToolResult<Output>>
```

The `call` method is where the actual work happens. It receives the validated input, a `ToolUseContext` that carries the full environment (abort controllers, file state caches, messages, app state), a permission callback, the assistant message that triggered it, and an optional progress callback for streaming status updates during long operations.

### Description and Prompt

```typescript
// src/Tool.ts (lines 387-393, 518-523)
description(input, options): Promise<string>
prompt(options): Promise<string>
```

Both `description` and `prompt` return text that gets woven into the system prompt. The `prompt` method generates the full tool documentation the model sees, while `description` provides a contextual summary. These are async because they may need to inspect the current working directory, permission context, or other runtime state.

### Safety Classification

```typescript
// src/Tool.ts (lines 402-416)
isConcurrencySafe(input: z.infer<Input>): boolean
isReadOnly(input: z.infer<Input>): boolean
isDestructive?(input: z.infer<Input>): boolean
interruptBehavior?(): 'cancel' | 'block'
```

These methods tell the orchestration layer how to handle the tool. `isConcurrencySafe` controls whether it can run in parallel with other tools. `isReadOnly` signals that it does not modify state. `isDestructive` marks irreversible operations. And `interruptBehavior` determines what happens when the user sends a new message while the tool is running: `'cancel'` aborts it; `'block'` lets it finish.

### Permission Checking

```typescript
// src/Tool.ts (lines 489-503)
checkPermissions(
  input: z.infer<Input>,
  context: ToolUseContext,
): Promise<PermissionResult>
```

Each tool gets to weigh in on whether a given invocation should be allowed, denied, or require a user prompt. This is the tool-specific layer of the permission system -- the general permission logic lives elsewhere, but `checkPermissions` lets individual tools add their own constraints.

### Result Mapping

```typescript
// src/Tool.ts (lines 557-559, 466)
mapToolResultToToolResultBlockParam(content: Output, toolUseID: string): ToolResultBlockParam
maxResultSizeChars: number
```

After execution, the raw tool output must be translated into the API's `ToolResultBlockParam` format -- what the model actually sees. The `maxResultSizeChars` property caps how large this result can be before it gets persisted to disk and replaced with a preview.

### Observable Input Backfill

```typescript
// src/Tool.ts (lines 481)
backfillObservableInput?(input: Record<string, unknown>): void
```

Some tools need to add derived or legacy fields to the input before observers (hooks, permissions, SDK streams) see it, without mutating the original input that feeds into `call()`. This hook lets tools do exactly that -- expand a relative path, add a computed field -- on a shallow clone.

## The `buildTool()` Factory

With 30+ methods on the interface, defining a tool from scratch would be painful. That is where `buildTool()` comes in:

```typescript
// src/Tool.ts (lines 757-792)
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,   // Assume not safe
  isReadOnly: (_input?: unknown) => false,            // Assume writes
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

The defaults are **fail-closed where it matters**: `isConcurrencySafe` defaults to `false` (assume parallel execution is unsafe), `isReadOnly` defaults to `false` (assume the tool writes). A tool author only overrides what they need.

## A Concrete Example: GrepTool

To see how this works in practice, look at the `GrepTool` in `src/tools/GrepTool/GrepTool.ts`:

```typescript
// src/tools/GrepTool/GrepTool.ts (lines 160-191)
export const GrepTool = buildTool({
  name: GREP_TOOL_NAME,
  searchHint: 'search file contents with regex (ripgrep)',
  maxResultSizeChars: 20_000,
  strict: true,

  async description() {
    return getDescription()
  },

  get inputSchema(): InputSchema {
    return inputSchema()
  },

  isConcurrencySafe() {
    return true       // Grep is read-only -- safe to run in parallel
  },
  isReadOnly() {
    return true       // No state mutation
  },

  async checkPermissions(input, context): Promise<PermissionDecision> {
    return checkReadPermissionForTool(GrepTool, input, /* ... */)
  },

  async call({ pattern, path, /* ... */ }, { abortController, getAppState }) {
    const results = await ripGrep(args, absolutePath, abortController.signal)
    // ... process and return results
    return { data: output }
  },

  mapToolResultToToolResultBlockParam(output, toolUseID) {
    // Format output for the model based on output_mode
    return { tool_use_id: toolUseID, type: 'tool_result', content: result }
  },
})
```

Notice what the GrepTool does *not* provide: `isEnabled`, `isDestructive`, `toAutoClassifierInput` (well, it overrides this one), `userFacingName` (it overrides this too). Everything it omits falls through to `buildTool`'s defaults.

:::info
The Tool interface has 30+ methods, but most have sensible defaults from `buildTool()`. A minimal tool only needs `name`, `call`, `inputSchema`, and `description`.
:::

## Key Takeaways

- **The `Tool` type is the universal contract** for every capability in the agent harness. It covers identity, execution, safety classification, permissions, and result formatting.
- **`buildTool()` provides fail-closed defaults** so tool authors only specify what makes their tool unique. Concurrency safety and read-only status default to the conservative choice.
- **Safety metadata is input-dependent.** Methods like `isConcurrencySafe(input)` take the actual input because some tools are safe for certain arguments but not others (e.g., a bash `ls` vs. `rm -rf`).
- **The `ToolResult` type can carry context modifiers** -- functions that update the `ToolUseContext` after execution. This is how tools can influence the environment for subsequent tool calls.
- **Description and prompt methods are async** because the documentation a tool exposes to the model may depend on runtime state like the current working directory or available permissions.
