# Agent Runtime Playbook

A diagram-first explainer for how an **Agentic CLI** (Codex, Claude Code, Open Code, Cursor, Copilot Chat, and the rest) drives an LLM, a provider, and a set of tools.

The point is the *workflow*. Model names and product names are placeholders.

---

## The one picture

If you only look at one thing in this repo, look at this:

![Agentic workflow as a directed graph](./diagrams/token-flow-graph.svg)

Five nodes, a few labeled edges. Input tokens flow right toward the model, output tokens flow back left, tool calls loop through the CLI, session state persists locally.

---

## The mental model in three lines

- **You** type a prompt.
- The **Agentic CLI** packages instructions + history + tools + your prompt into a request, sends it to a **Provider**, which routes it to a **Model**.
- The model emits tokens one at a time; the CLI streams them back, optionally runs tools, and loops until the answer is done.

That's it. Everything else in this repo is detail on one of those arrows.

---

## What "Agentic CLI" covers

Different products, same shape:

| Tool | Vendor |
|---|---|
| Codex CLI | OpenAI |
| Claude Code | Anthropic |
| Open Code | open source |
| Cursor / Copilot Chat | IDE-embedded variants |

They all do the same job: orchestrate prompts, manage context, run tools locally, talk to a remote model. When this repo says "the CLI does X," it means any of them.

---

## A walk through the system, one diagram at a time

### 1. The request path

![CLI request flow](./diagrams/cli-request-flow.svg)

Local on the left, remote on the right. Your prompt becomes a request payload, the provider authenticates and routes it, the model decodes tokens, the stream comes back.

### 2. The tool loop

![Tool-calling loop](./diagrams/agent-tool-loop.svg)

The model doesn't open files or run shell commands. It *asks* the CLI to do those things by emitting a structured tool call. The CLI executes the tool and feeds the result back as more input on the next model turn.

### 3. Inside the model

![Model decoding loop](./diagrams/model-decoding-loop.svg)

One inference call is itself a loop: tokenize → embed → transformer layers → pick next token → append → repeat. The answer doesn't appear at once; it's decoded one token at a time.

### 4. A full turn, as a sequence

![Engineer sequence flow](./diagrams/engineer-sequence-flow.svg)

When tools are involved, a single user turn spans multiple request-response hops. This shows the whole choreography.

---

## Reading order

Two short files cover everything most people need:

1. **[workflow_overview.md](./workflow_overview.md)** — the concepts behind the diagrams, in plain English.
2. **[workflow_glossary.md](./workflow_glossary.md)** — one-line definitions for `token`, `context window`, `inference`, etc.

---

## Going deeper (optional)

If you are building, operating, or debugging one of these systems, the `deep_dives/` folder has implementation-level material:

- **[workflow_engineer_view.md](./workflow_engineer_view.md)** — same model, written for engineers (KV cache, context window engineering, latency drivers).
- **[deep_dives/request_trace.md](./deep_dives/request_trace.md)** — one full turn traced end-to-end with concrete payloads.
- **[deep_dives/schema_patterns.md](./deep_dives/schema_patterns.md)** — provider-agnostic request/response shapes (messages, tools, streaming, stop reasons, usage).
- **[deep_dives/debugging_runbook.md](./deep_dives/debugging_runbook.md)** — failure patterns and how to diagnose which layer broke.

These are reference material. You do not need them to understand the diagrams.
