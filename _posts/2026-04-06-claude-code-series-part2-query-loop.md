---
layout: post
title: "Claude Code Architecture Series (Part 2): The Query Loop & Tool Orchestration"
date: 2026-04-06
permalink: /engineering-strategy/claude-code-query-loop-orchestration/
tags: [Async Programming, Event Loops, LLM Tooling, System Architecture]
author: Novi Develop
description: "Part 2 of our Claude Code Architecture series. We break down the Query Engine's asynchronous event loop and the intricate tool orchestration pipeline."
---

In [Part 1](/engineering-strategy/claude-code-architecture-startup/), we explored how Claude Code boots up almost instantaneously via parallel execution. But what happens after the terminal prompt appears and a user hits *Enter*? 

In Part 2, we dissect the absolute heart of the application: **The Query Loop** and its associated **Tool System Orchestration**.

## The Query Loop: An Asynchronous Generator

When you submit a message, you interact with `QueryEngine.ts`, which wraps the `queryLoop()` generator function. This function sits between the Ink Presentation layer and the external LLM APIs, handling complex streaming API scenarios.

### Step-by-Step Flow:
1. **Prepare Context:** The system dynamically concatenates instructions, including base system prompts, `claude.md` file hints, Git state, context bounds, and the necessary Zod tool schemas.
2. **Streaming Yield:** An HTTP request hits the Claude API with `stream: true`. Using `for await (const event of stream)`, the underlying generator function yields parts of strings or reasoning blocks natively to the React `<Messages />` component.
3. **Execution Decision:** Does the API response contain `tool_use` blocks? 
    *   **No:** The operation concludes with `{ reason: 'completed' }` and hands control back to the user prompt.
    *   **Yes:** The loop pauses streaming and dives into Tool Execution.

## Tool Execution Orchestration

If Claude wants to perform file IO or hit the web via tools, it drops into `toolOrchestration.ts`. This phase handles parallelization and security.

### 1. Verification and Security Layers
Before execution, every tool (`BashTool`, `FileEdit`, etc.) passes through intense scrutiny:
*   **Schema Validation:** Zod validates that the input payload sent by the LLM is perfectly shaped.
*   **Static Permission Mapping:** Checks the local configuration settings (e.g., is auto-edit allowed?).
*   **Interactive Checks:** If a command exceeds an auto-run heuristic, the engine yields a status pause causing a permission modal to present itself in the UI.

### 2. Execution and Concurrency
A major performance characteristic of the system is identifying tool mutability.
If Claude returns multiple `tool_use` references, the orchestrator divides them:
*   *Read-Only Tools* (like `GrepTool`, `GlobTool`) are executed simultaneously using parallel task batching.
*   *Write Tools* (like `BashTool`, `FileWrite`) are strictly serialized to avert race condition corruption across your code base.

### 3. Hook Firing and Loop Back
The completion of a tool isn't the end of the line. The system invokes lifecycle methods like `PreToolUse` and `PostToolUse` for observability. Once all tools finish resolving, the results are formatted natively as `tool_result` messages. 

The query loop then forcefully invokes *itself*, taking those tool outputs and transmitting them back to the Claude endpoint in a recursive loop until the agent decides the overarching request has been fulfilled.

---

In the next part, we'll follow a message closely through an end-to-end trace from the input capture straight into long-term infrastructure persistence.
