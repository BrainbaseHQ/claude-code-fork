---
sidebar_position: 0
slug: /intro
title: Introduction
---

# Anatomy of a Production Agent Harness

An agent harness, at its core, manages a conversation between a language model and a set of tools. The basic structure — call the model, execute any requested tools, repeat — is straightforward enough to implement in an afternoon.

The interesting engineering emerges when this loop needs to run reliably across sessions lasting hours, handle failures gracefully at every level, remain responsive to the user throughout, and do all of this while managing a finite context window that the conversation steadily fills.

This course is a deep, structured study of how these problems are solved in practice. We use the **Claude Code** source code as our reference implementation — not because it's the only way to build an agent harness, but because it's a mature, production system that has encountered (and addressed) most of the problems you'll face when building your own.

## Who This Course Is For

You should be comfortable with:

- **TypeScript/JavaScript** and async programming (`async/await`, Promises)
- **The basic agent concept**: an LLM that can request tool executions (file reads, shell commands, API calls), receive the results, and continue reasoning
- **The Anthropic Messages API** at a conceptual level (system prompts, user/assistant message alternation, tool_use/tool_result blocks)

You do **not** need to have built a production agent system before. That's what we're here to learn.

## What You'll Learn

The course is organized into six modules, each building on the previous:

### [1. Foundations](./01-foundations/01-production-vs-toy.md)
What separates a production harness from a prototype. The core data models — messages, tools, context — that everything else is built on. And AsyncGenerators, the primitive that makes streaming possible.

### [2. The Query Loop](./02-query-loop/01-loop-structure.md)
The central `while(true)` loop that drives the agent. How it manages state across iterations, streams API responses, executes tools, and decides whether to continue or stop — including seven distinct reasons to loop again.

### [3. Tool System](./03-tool-system/01-tool-interface.md)
How tools are defined, dispatched, and executed. The concurrency model that safely runs read-only tools in parallel. The StreamingToolExecutor that overlaps tool execution with model output. And the permission system that gates which tools can run.

### [4. Context Window Management](./04-context-window/01-the-problem.md)
The finite context window is the hard constraint of every agent system. Five layers of defense — from cheap result truncation to full conversation summarization — that keep the conversation within bounds. Plus the withholding pattern for transparent error recovery.

### [5. Subagents](./05-subagents/01-why-subagents.md)
When a single agent isn't enough. Three execution models for subagents (synchronous, background, bubble), the resource lifecycle that prevents leaks, and a prompt cache optimization that makes parallel subagents dramatically cheaper.

### [6. Advanced Patterns](./06-advanced/01-abort-cancellation.md)
Abort and cancellation with WeakRef-based propagation trees. The 5,000-line REPL component that bridges imperative execution with declarative rendering. The hooks system for extensibility. And memory management patterns for long-running sessions.

## How to Read This Course

Each page is designed to be read in 5-8 minutes. Code examples are drawn from the actual Claude Code source, with file path references so you can explore further. Mermaid diagrams illustrate control flow and data relationships.

The modules are sequential — later modules reference concepts from earlier ones. If you're already familiar with a topic, the **Key Takeaways** at the bottom of each page will tell you whether there's something new for you.

:::tip Transferable Patterns
While we use Claude Code as our reference, the patterns here — state machine loops, streaming concurrency, context compaction, resource lifecycle management — apply to any production agent system. Each page's Key Takeaways section highlights what's transferable.
:::

## Source Code Reference

The Claude Code source lives alongside this course. Key files you'll encounter throughout:

| File | Role |
|-|-|
| `src/query.ts` | The main query loop |
| `src/Tool.ts` | Tool interface and types |
| `src/services/tools/StreamingToolExecutor.ts` | Streaming tool concurrency |
| `src/services/tools/toolOrchestration.ts` | Tool batching and dispatch |
| `src/tools/AgentTool/AgentTool.tsx` | Agent spawning |
| `src/tools/AgentTool/runAgent.ts` | Agent execution lifecycle |
| `src/tools/AgentTool/forkSubagent.ts` | Fork optimization for prompt cache |
| `src/services/compact/autoCompact.ts` | Context compaction |
| `src/screens/REPL.tsx` | UI orchestration |
| `src/utils/hooks.ts` | Hook system |
| `src/utils/abortController.ts` | Cancellation primitives |

---

Ready to begin? Start with [What Makes a Harness Production-Grade](./01-foundations/01-production-vs-toy.md).
