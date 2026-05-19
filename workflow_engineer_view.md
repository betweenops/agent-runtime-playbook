# Workflow: Engineer View

Same workflow as [workflow_overview.md](./workflow_overview.md), but written for engineers who want the implementation tradeoffs.

The product names are still placeholders. The mechanics are the point.

Companion files in `deep_dives/`:

- [request_trace.md](./deep_dives/request_trace.md) — one turn traced end-to-end with concrete payloads
- [schema_patterns.md](./deep_dives/schema_patterns.md) — provider-agnostic request/response shapes
- [debugging_runbook.md](./deep_dives/debugging_runbook.md) — failure patterns by layer

![Engineer sequence flow](./diagrams/engineer-sequence-flow.svg)

## System boundary

Three layers, kept separate or debugging gets painful:

| Layer | Owns | Cares about |
|---|---|---|
| **Agentic CLI** (local) | prompt capture, context assembly, tool execution, streaming UX, session state | file access, tool permissions, retries, persistence |
| **API Provider** (remote) | auth, rate limits, validation, routing, metering, streaming transport | quotas, caching, usage accounting |
| **Model** (remote) | tokenization, forward pass, decoding | latency, context length, KV cache, output quality |

## Request lifecycle

For one turn, the CLI usually does:

1. Accept the user message.
2. Resolve system + developer instructions.
3. Load prior conversation state.
4. Attach relevant tool results, retrieved docs, file excerpts.
5. Serialize into the provider's request schema.
6. Authenticate and send.
7. Receive a stream — text or tool call.
8. If tool: execute, send result back as a follow-up turn.
9. Persist session state.

Conceptually:

```text
turn_n_request =
    instructions
  + relevant_history
  + latest_user_input
  + tool_results_if_any
  + retrieval_context_if_any
```

For a concrete walkthrough, see [request_trace.md](./deep_dives/request_trace.md).

## What the CLI actually owns

The CLI isn't doing inference. It's doing orchestration:

- model + provider selection
- how much history to include
- truncation and summarization of old context
- tool schema formatting
- shell/file tool execution
- retries and failure handling
- incremental rendering
- on-disk session state

Most "agent behavior" is a property of CLI design, not raw model capability.

## Representative request shape

Exact schema depends on the provider, but the shape is usually:

```json
{
  "model": "example-model",
  "instructions": "system or developer guidance",
  "input": [
    {"role": "user", "content": "Explain this stack trace"},
    {"role": "tool", "content": "Contents of ./logs/app.log"},
    {"role": "user", "content": "Now suggest a fix"}
  ],
  "tools": [{"name": "read_file", "description": "Read a local file"}],
  "stream": true
}
```

The "conversation" is structured input for the current request. Memory is reconstructed state, not a hidden mental model.

For more on schemas, see [schema_patterns.md](./deep_dives/schema_patterns.md).

## Why KV cache matters

Inside one request, transformer inference reuses cached key/value tensors for already-processed tokens. Generating token *n+1* is much cheaper than recomputing the whole sequence.

Provider-side **cached tokens** in the usage block are related but different — they refer to billing/serving optimizations for repeated prompt prefixes *across* requests, not the model's internal per-request KV cache.

## Tool calling is a control loop

![Tool-calling loop](./diagrams/agent-tool-loop.svg)

```text
user input
  → CLI sends context to model
  → model requests tool OR emits answer
  → CLI executes tool
  → CLI sends result back as next turn
  → model continues
  → CLI returns final answer
```

The model emits structured output the CLI interprets as a tool request. The model never touches your filesystem or shell directly.

## Context window engineering

The biggest lever. Typical contents:

- system + developer instructions
- recent chat history
- retrieved docs
- file excerpts
- tool outputs
- the latest user message

Tradeoffs:

- more context → better accuracy, but more cost + latency
- irrelevant context → worse quality
- long tool outputs crowd out useful tokens
- summarization shrinks size but loses detail

Many "the model is dumb" complaints are context engineering bugs.

## Latency and cost drivers

- input token count (biggest hit on time-to-first-token)
- output token count
- model size and serving tier
- tool round-trips
- retrieval round-trips
- streaming overhead
- provider-side queuing or rate limits

Time-to-first-token usually reflects prompt processing and prefill cost.

## Why the system feels stateful

The model is stateless across requests. The CLI fakes statefulness by:

1. persisting prior interaction state
2. replaying or summarizing it into the next request

This explains common failures:

- old facts vanish because they fell out of context
- behavior changes when instruction order changes
- tool outputs only influence later turns if they were preserved

## Failure modes engineers actually hit

- **Context overflow** — too much history pushes critical info out
- **Instruction collisions** — system/developer/user instructions conflict
- **Tool schema drift** — CLI expects one format, model emits another
- **Retrieval pollution** — low-quality retrieved docs add noise
- **Non-deterministic outputs** — sampling or model updates shift results
- **Streaming misinterpretation** — partial stream treated as final

Triage order: payload → instructions → included history → tool serialization → retrieval quality → token limits/truncation → finally, model reasoning. Many "LLM bugs" are packaging bugs. Full patterns in [debugging_runbook.md](./deep_dives/debugging_runbook.md).

## Minimum useful observability

- raw request payload (secrets redacted)
- effective composed instructions
- input + output token counts
- truncation / summarization events
- tool call requests and results
- time to first token
- total turn latency
- provider errors, retries, rate-limit events

Without these, debugging is guesswork.

## Token usage categories

- `input` — everything sent in
- `output` — everything generated
- `cached` — reused prompt prefix (provider-dependent)
- `reasoning` — extra internal inference budget (provider-specific)

Not standardized across providers. Read as provider-defined accounting.

## Operational notes

- the API key grants billable service access
- local tool execution can leak files, secrets, or infra if poorly scoped
- session storage may contain sensitive prompts, outputs, and tool traces
- retrieval can surface private data into model context

The model is the visible part. The data path is where most operational risk lives.

## Takeaway

- model = probabilistic token generator
- provider = managed serving and accounting
- Agentic CLI = orchestration that makes the workflow useful

The highest-leverage places to improve behavior are context assembly, tool design, retrieval quality, and request/response tracing.
