---
layout: post
title: "Claude Code Architecture (Part 3): End-to-End Context & Memory Subsystems"
date: 2026-04-06
permalink: /engineering/claude-code-context-memory-subsystems/
tags: [Data Persistence, Memory Management, LLM Scaling, System Design]
author: Novi Develop
description: "The final part of our NoviDevelop architecture series. Learn how Claude calculates token budgets and manages context state persistently."
---

In our final [NoviDevelop](/engineering/) architectural deep dive (following [Part 2](/engineering/claude-code-async-query-loop/)), we uncover the most critical backend system behind modern coding UI: **Context Strategy**.

Sending millions of repository file text strings into an LLM context window quickly exceeds hard processing limitations without stringent optimization logic. How does Claude scale infinitely?

## Constant Token Tracking

The file `src/services/tokenEstimation.ts` demonstrates a robust multi-pass heuristic. 
Since exact API token calculation causes latency overhead, Claude implements a fallback mechanism estimating "1 token per 4 characters." This combined heuristic tracks the conversation state to establish a "Warning Threshold."

The architecture reserves exactly 20,000 tokens as a safety-net parameter preventing out-of-memory cascades.

## The Tri-Layered Pipeline

Prior to executing network inference calls, every message passes through a modification pipeline:
1.  **Tool Result Thresholds:** If the raw execution logs of a command exceed 20,000 characters, the data buffers are pruned and written securely to disk, and replaced inline with a `[saved to disk]` placeholder link.
2.  **HISTORY_SNIP Hooks:** If previous tool queries are over N turns old, the architecture runs a background replacement script dropping textual content but preserving the `tool_use` framing to keep the LLM logically grounded.
3.  **Autocompaction Mechanisms:** The absolute failover. If the context expands within 13,000 tokens of the absolute limit, an independent Background Process queries the entire log, drafts an overarching Session Summary, persists it in `.claude/session_memory` on disk, and seamlessly clears the UI slate.

This three-tiered strategy allows software engineers to leave the backend terminal application running across thousands of files indefinitely without encountering hard limit errors.

This concludes our 3-part framework architectural breakdown!
