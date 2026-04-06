---
layout: post
title: "Claude Code Architecture Series (Part 3): The Message Lifecycle Walkthrough"
date: 2026-04-06
permalink: /engineering-strategy/claude-code-message-lifecycle/
tags: [Message Queues, Application State, Data Persistence, React]
author: Novi Develop
description: "Part 3 of our series traces the lifecycle of a single prompt from the terminal UI down to API execution and long-term storage."
---

Welcome to Part 3 of our Claude Code Architecture series. In [Part 2](/engineering-strategy/claude-code-query-loop-orchestration/), we detailed the synchronous and asynchronous nature of the Query Loop. Continuing our technical evaluation, we will now follow a single prompt on an end-to-end journey from the presentation layer to stable infrastructure persistence.

When a user sits at the command line and types a prompt in Claude Code v2.1.88, multiple architectural layers synchronize smoothly.

## Step 1: Input Capture (Presentation Layer)

Your environment is an instance of the `REPL.tsx` component managed by a custom Ink Renderer. When you type and submit your entry, the React components intercept it. A global `AppStateStore` captures the submission using React's contextual providers and records the start of a transcript event. 

## Step 2: Message Creation (Application Layer)

The prompt transitions out of the UI into the `QueryEngine.ts`. The input assumes a strict representation mimicking the Anthropic Chat API format (e.g. `role: user`, `content: ...`). The application records this to the session transcript, establishing an in-memory conversational log. 

## Step 3 & 4: Context Gathering & API Calling (Infrastructure)

The system invokes its dependency layers immediately:
1. **Context Compaction Checks:** Is the context token cache nearing its threshold? If yes, it prunes background messages (a concept we'll explore thoroughly in Part 5).
2. **Payload Construction:** The payload groups your intent with context objects (git paths, date data, and system prompts).
3. **Execution:** The application issues a POST request, primarily routing through the Anthropic or Bedrock endpoints, maintaining a persistent TCP socket for rapid return execution.

## Step 5: Streaming the Response (App -> Presentation)

Because the Query Loop is an async generator, each byte returned by the model generates a `yield` event. As the stream resolves, React hooks observing the memory model recalculate to re-render the terminal window inside the `<VirtualMessageList>` component. You perceive immediate visual feedback typing out your text in the console.

## Step 6: Recalculating Tool Constraints

If Claude Code decides to invoke a tool (like `FileRead`), the system suspends output generation, jumps into `toolOrchestration.ts`, resolves the read contents, outputs `{ reason: 'completed' }` for the tool sub-run, and cycles backward.

## Step 7 & 8: Loop Back & Completion

After sending the retrieved tool data back to the inference layer, the model finally produces its deterministic output solution. The `queryLoop()` detects the end state and terminates. The spinner in the terminal stops, the line indicator updates token costs utilizing `cost-tracker.ts`, and the terminal is returned completely to the human user.

## Step 9: Post-Turn Persistence 

Before the software completely suspends operation and waits for the next command, background processes continue to run:
*   Memory structures are stored back down into JSONL persistence files.
*   Token expenditures report back to Datadog telemetry or local storage.
*   A "task" sub-routine evaluates if your interaction signifies that you completed a defined checklist item and naturally ticks it off in internal state stores.

---

In Part 4, we will shift focus completely. Moving away from the architectural flows, we will dive specifically into the engineering magic behind **Agentic Search** and how Claude uses `ripgrep` securely on massive repositories without exploding context windows.
