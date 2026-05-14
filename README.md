# Codex / LLM Workflow Docs

This directory is a guided documentation set for understanding how a Codex-style agent workflow interacts with an LLM, an API provider, tools, local session state, and request/response schemas.

The examples use model names only as placeholders. The important part is the workflow.

## Start Here

If you are new to the topic, read these in order:

1. [workflow_overview.md](./workflow_overview.md)
2. [workflow_glossary.md](./workflow_glossary.md)
3. [workflow_engineer_view.md](./workflow_engineer_view.md)

If you are already technical and want the implementation view first:

1. [workflow_engineer_view.md](./workflow_engineer_view.md)
2. [engineer_request_trace.md](./engineer_request_trace.md)
3. [engineer_debugging_runbook.md](./engineer_debugging_runbook.md)
4. [provider_agnostic_schema_patterns.md](./provider_agnostic_schema_patterns.md)

## What Each File Does

### Core Overview

- [workflow_overview.md](./workflow_overview.md): the general explainer for how prompts move through Codex, the provider, the model, and any tool loop
- [workflow_glossary.md](./workflow_glossary.md): short definitions for recurring terms such as `context window`, `inference`, `streaming`, and `tool call`

### Engineering View

- [workflow_engineer_view.md](./workflow_engineer_view.md): the technical architecture view, including system boundaries, request lifecycle, KV cache, latency, and observability
- [engineer_request_trace.md](./engineer_request_trace.md): a concrete traced request from prompt to tool call to final answer
- [engineer_debugging_runbook.md](./engineer_debugging_runbook.md): a practical troubleshooting guide for wrong answers, tool failures, truncation, latency, retrieval noise, and other common problems
- [provider_agnostic_schema_patterns.md](./provider_agnostic_schema_patterns.md): provider-neutral request/response patterns for chat, tools, streaming, stop reasons, and usage accounting

### Diagrams

- [codex-request-flow.svg](./diagrams/codex-request-flow.svg): local runtime vs provider vs model
- [agent-tool-loop.svg](./diagrams/agent-tool-loop.svg): how tool-calling changes the workflow
- [model-decoding-loop.svg](./diagrams/model-decoding-loop.svg): what happens inside inference at a high level
- [engineer-sequence-flow.svg](./diagrams/engineer-sequence-flow.svg): sequence-style engineer view of a multi-step turn

## Recommended Reading Paths

### Path 1: General Understanding

Use this if the goal is to understand the stack conceptually without getting stuck in implementation detail.

1. [workflow_overview.md](./workflow_overview.md)
2. [workflow_glossary.md](./workflow_glossary.md)

### Path 2: How the System Actually Works

Use this if the goal is to understand how the runtime, provider, model, tools, and session state fit together operationally.

1. [workflow_engineer_view.md](./workflow_engineer_view.md)
2. [engineer_request_trace.md](./engineer_request_trace.md)
3. [provider_agnostic_schema_patterns.md](./provider_agnostic_schema_patterns.md)

### Path 3: Debugging and Production Thinking

Use this if the goal is to diagnose bad outputs or design better instrumentation.

1. [engineer_debugging_runbook.md](./engineer_debugging_runbook.md)
2. [engineer_request_trace.md](./engineer_request_trace.md)
3. [provider_agnostic_schema_patterns.md](./provider_agnostic_schema_patterns.md)

## Why `workflow_overview.md` Still Exists

It is still the best single-file entry point for the full topic.

The newer files split the content by audience and use case:

- overview
- glossary
- engineer architecture
- traced request
- debugging
- schema patterns

That split reduces cognitive load without removing the general explainer.

## Suggested Next Additions

If this doc set keeps growing, the next useful documents would likely be:

- a security and data-handling guide for prompts, tools, secrets, and session state
- a retrieval-specific guide covering chunking, ranking, and source quality
- a testing guide for agent workflows and tool loops
