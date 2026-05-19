# Provider-Agnostic Schema Patterns

This file shows common request and response patterns used in Agentic CLI workflows. Examples are provider-agnostic; the same shapes show up whether you are looking at Codex, Claude Code, Open Code, Cursor, or any other agent-style client.

These examples are intentionally provider-agnostic. Real APIs differ in naming and exact nesting, but the structural ideas are usually similar.

## Why This Matters

Many workflow misunderstandings come from collapsing all LLM interaction into:

```text
prompt -> answer
```

In practice, the wire format often includes:

- model selection
- instruction layers
- structured input messages
- tool schemas
- streaming events
- usage metadata
- explicit stop reasons

If you understand the schema shape, debugging and system design get easier.

## Canonical Request Categories

Most production workflows can be reduced to a few request types:

1. plain text generation
2. multi-message conversation
3. tool-enabled generation
4. retrieval-augmented generation
5. streaming generation

These are often variations of the same base payload.

## Base Request Shape

A generic request usually contains:

```json
{
  "model": "example-model",
  "instructions": "high-priority behavior guidance",
  "input": [
    {
      "role": "user",
      "content": "Explain this error"
    }
  ],
  "stream": false
}
```

Typical fields:

- `model`: which model or deployment to use
- `instructions`: system or developer guidance
- `input`: the current turn payload, often as structured messages
- `stream`: whether tokens should arrive incrementally

Some providers split `instructions` across `system`, `developer`, or other role-specific fields. Others collapse them into the message list.

## Conversation Request Pattern

For multi-turn chat, the payload typically includes reconstructed prior state:

```json
{
  "model": "example-model",
  "input": [
    {
      "role": "system",
      "content": "Be accurate and concise."
    },
    {
      "role": "user",
      "content": "What is a Kubernetes operator?"
    },
    {
      "role": "assistant",
      "content": "A Kubernetes operator is..."
    },
    {
      "role": "user",
      "content": "How is that different from a controller?"
    }
  ]
}
```

The important point is that the model usually sees a rebuilt message history, not hidden memory.

## Tool-Enabled Request Pattern

When tools are available, the request usually exposes a tool schema or manifest:

```json
{
  "model": "example-model",
  "instructions": "Use tools before making claims about local files.",
  "input": [
    {
      "role": "user",
      "content": "Check the startup log and explain the crash."
    }
  ],
  "tools": [
    {
      "name": "read_file",
      "description": "Read a file from the local workspace",
      "parameters": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string"
          }
        },
        "required": ["path"]
      }
    }
  ],
  "stream": true
}
```

The tool schema is part of the prompt surface. Better schemas usually lead to better tool selection.

## Retrieval-Augmented Request Pattern

When external documents are injected, they often appear as additional context messages:

```json
{
  "model": "example-model",
  "input": [
    {
      "role": "system",
      "content": "Answer using the provided documentation when relevant."
    },
    {
      "role": "context",
      "content": "Document excerpt: The deployment controller reconciles desired state..."
    },
    {
      "role": "user",
      "content": "Why did this rollout stall?"
    }
  ]
}
```

Some systems use a dedicated `context`, `documents`, or `attachments` field instead of injecting retrieval results as messages. The underlying idea is the same: external text is added to the model's working context.

## Tool Call Response Pattern

When the model wants a tool, the response is often structured rather than plain text.

Conceptually:

```json
{
  "output": [
    {
      "type": "tool_call",
      "name": "read_file",
      "arguments": {
        "path": "./logs/app.log"
      }
    }
  ],
  "stop_reason": "tool_call"
}
```

Important detail: this is not the tool being executed. This is the model requesting that the runtime execute it.

## Tool Result Follow-Up Pattern

After the runtime executes the tool, the next request includes the result:

```json
{
  "model": "example-model",
  "input": [
    {
      "role": "user",
      "content": "Check the startup log and explain the crash."
    },
    {
      "role": "tool",
      "name": "read_file",
      "content": "Error: listen EADDRINUSE: address already in use :::3000"
    }
  ]
}
```

This is the mechanism that turns tool output into model-visible context.

## Final Text Response Pattern

When the model is ready to answer directly, the response often looks conceptually like:

```json
{
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "content": "The service is crashing because port 3000 is already in use."
    }
  ],
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 812,
    "output_tokens": 96
  }
}
```

Provider field names differ, but the common elements are:

- generated content
- termination reason
- usage accounting

## Streaming Event Pattern

Streaming APIs usually emit multiple events instead of one final blob.

Conceptually:

```json
{"type":"response.started","id":"resp_123"}
{"type":"output_text.delta","delta":"The service "}
{"type":"output_text.delta","delta":"is crashing "}
{"type":"output_text.delta","delta":"because port 3000 "}
{"type":"output_text.delta","delta":"is already in use."}
{"type":"response.completed","stop_reason":"end_turn","usage":{"input_tokens":812,"output_tokens":96}}
```

The exact event names vary, but most streaming systems expose some combination of:

- response start
- token or text deltas
- tool call events
- completion event
- final usage metadata

## Stop Reason Pattern

Good APIs usually expose why generation stopped.

Common categories:

- `end_turn`
- `tool_call`
- `max_output_tokens`
- `stop_sequence`
- `content_filter`
- `error`

This field is operationally important. It helps distinguish a complete answer from an interrupted or truncated one.

## Usage Metadata Pattern

Many providers attach usage information to the response:

```json
{
  "usage": {
    "input_tokens": 812,
    "cached_tokens": 640,
    "output_tokens": 96,
    "reasoning_tokens": 128,
    "total_tokens": 1036
  }
}
```

Not all providers expose all categories, and definitions can differ.

Treat these as provider accounting fields, not universal architecture primitives.

## Common Role Patterns

Typical message roles include:

- `system`
- `developer`
- `user`
- `assistant`
- `tool`
- `context`

Not every provider supports every role explicitly. Some normalize everything into a simpler list, while others add extra structured types.

## Two Common Design Styles

### Message-Centric Style

Everything is a message:

```json
{
  "messages": [
    {"role": "system", "content": "Be concise."},
    {"role": "user", "content": "Explain the error."}
  ]
}
```

### Structured Input Style

Instructions, input items, tools, and streaming are first-class top-level fields:

```json
{
  "instructions": "Be concise.",
  "input": [
    {"role": "user", "content": "Explain the error."}
  ],
  "tools": [],
  "stream": true
}
```

Both styles can represent similar workflows.

## What to Log for Each Request

If you want schema-level observability, log at least:

- request id
- model id
- effective instructions
- final serialized input
- exposed tools
- stop reason
- usage block
- latency metrics

If the system is streaming, also log:

- stream start
- tool call events
- completion event

## Schema-Level Failure Modes

Bad behavior often comes from schema issues rather than pure reasoning issues.

Common examples:

- tool schema missing required fields
- tool output inserted with the wrong role
- retrieved documents placed in a field the model does not actually read
- truncation dropping high-priority instructions
- partial stream treated as final response
- provider stop reason ignored by the runtime

These bugs are easy to miss because the final symptom still looks like "bad AI output."

## Practical Takeaway

Provider APIs differ in syntax, but most agent workflows reuse the same small set of concepts:

- instructions
- structured input
- tool definitions
- tool call outputs
- streamed deltas
- stop reasons
- usage metadata

If you understand those patterns, you can usually map any provider's schema back to the same runtime-model-tool workflow.
