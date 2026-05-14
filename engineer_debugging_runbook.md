# Engineer Debugging Runbook

This runbook is for diagnosing bad behavior in a Codex-style agent workflow.

The goal is not to ask "Did the model mess up?" too early.

The goal is to determine which layer failed:

- runtime packaging
- tool execution
- retrieval or context assembly
- provider behavior
- model reasoning

## First Principle

Do not debug from the final answer alone.

For any problematic turn, capture at least:

1. the exact user request
2. the effective instructions
3. the final payload sent to the provider
4. any tool calls emitted by the model
5. the actual tool outputs returned to the model
6. token counts and latency data
7. the final streamed answer

Without this trace, most debugging turns into speculation.

## Fast Triage Questions

Start with these:

1. Did the model have the information it needed?
2. Was the right information included in the context window?
3. Did the runtime execute the right tool with the right arguments?
4. Did the tool output make it back into the next model turn?
5. Was the answer wrong because of reasoning, or because the model never saw the right facts?

That distinction usually tells you where to look next.

## Canonical Debug Order

Inspect the turn in this order:

1. user request
2. effective instructions
3. included history and retrieved context
4. serialized provider payload
5. tool call request from the model
6. tool execution result
7. follow-up payload after tool execution
8. final answer
9. token usage and latency

This sequence mirrors the actual system flow. Debugging in a different order often hides the real fault.

## Failure Pattern: The Answer Ignores an Important Fact

### Symptom

The final answer contradicts something that should have been obvious from prior conversation, a file, or a tool result.

### Likely Causes

- the fact was not included in the current context window
- the relevant history was truncated or summarized poorly
- the tool result never made it back into the follow-up request
- irrelevant retrieved context diluted the important signal

### What to Check

1. Search the exact provider payload for the missing fact.
2. Check whether truncation removed it.
3. Check whether a summarizer rewrote it incorrectly.
4. Check whether the fact existed only in local state, not in the actual request.

### Typical Fixes

- prioritize high-value state in context assembly
- reduce noisy history or retrieval output
- preserve critical tool outputs verbatim
- add explicit tests around truncation behavior

## Failure Pattern: The Model Uses the Wrong Tool

### Symptom

The model chooses an irrelevant tool, uses the right tool with bad arguments, or answers without using a tool when it should have.

### Likely Causes

- tool descriptions are ambiguous
- the runtime exposed too many overlapping tools
- instructions about tool use are weak or conflicting
- the model inferred the wrong path from incomplete context

### What to Check

1. Inspect the tool schema and descriptions exposed in the request.
2. Check whether tool names are too similar.
3. Check whether instructions clearly say when tool use is required.
4. Compare the emitted tool arguments against the user's actual request.

### Typical Fixes

- tighten tool descriptions
- remove redundant tools from the exposed set
- require tool use before making claims about external state
- validate tool arguments in the runtime before execution

## Failure Pattern: Tool Output Exists but the Final Answer Is Still Wrong

### Symptom

The runtime executed the right tool, but the final answer still misses or misuses the result.

### Likely Causes

- tool output was truncated before being returned
- tool output formatting was confusing
- the follow-up payload was malformed
- the model had too much competing context
- the model made an actual reasoning error

### What to Check

1. Verify the exact tool output that was inserted into the follow-up request.
2. Compare the raw tool output with the final answer.
3. Check whether the follow-up request preserved the relevant lines.
4. Check whether additional context distracted the model from the useful signal.

### Typical Fixes

- bound and format tool output more clearly
- highlight the relevant slice of large outputs
- reduce unrelated context in the follow-up turn
- add runtime-side postprocessing for large logs or traces

## Failure Pattern: The Answer Changes Unpredictably Across Similar Runs

### Symptom

Two nearly identical requests produce meaningfully different answers.

### Likely Causes

- non-deterministic decoding
- provider-side model updates
- small prompt or ordering changes
- different retrieval results
- hidden state differences in assembled context

### What to Check

1. Diff the full provider payloads, not just the user prompt.
2. Compare token counts and included history.
3. Compare retrieval results and tool outputs.
4. Check whether the provider model version or serving tier changed.

### Typical Fixes

- reduce unnecessary prompt variance
- stabilize retrieval ranking
- snapshot or pin model versions when possible
- make downstream systems robust to bounded output variation

## Failure Pattern: The Response Is Slow

### Symptom

The turn works, but latency is too high.

### Likely Causes

- very large input context
- slow tool execution
- multiple tool round-trips
- provider queueing or rate limits
- slow first-token latency on a larger model tier

### What to Check

1. Measure time to first token separately from total turn time.
2. Measure time spent in tool execution separately from model time.
3. Compare input token size across fast and slow runs.
4. Check whether repeated provider retries are occurring.

### Typical Fixes

- shrink the prompt and history window
- reduce large tool payloads
- cache retrieval or tool results where safe
- split long workflows into smaller turns
- choose a lower-latency model when quality allows

## Failure Pattern: The Answer Stops Early or Feels Truncated

### Symptom

The model begins correctly, then stops before completing the task.

### Likely Causes

- output token limit reached
- runtime prematurely treated a partial stream as final
- provider interruption or timeout
- tool loop ended before the final answer step

### What to Check

1. Inspect output token limits for the request.
2. Check stream termination handling in the runtime.
3. Check whether the provider returned an explicit stop reason.
4. Confirm that all expected tool turns completed.

### Typical Fixes

- increase output allowance when justified
- fix stream completion handling
- distinguish partial from terminal events
- add watchdogs around multi-step tool loops

## Failure Pattern: The Model Hallucinates Local Facts

### Symptom

The answer makes confident claims about files, commands, or system state that were never actually checked.

### Likely Causes

- instructions did not require tool verification
- the runtime exposed tools but did not encourage or require them
- the model was allowed to answer from priors alone

### What to Check

1. Inspect whether the model had any real local-state evidence.
2. Check whether tool use was available but unused.
3. Check the instruction hierarchy for requirements like "use tools before claiming file contents."

### Typical Fixes

- require evidence-backed answers for local-state questions
- enforce tool use for file or shell assertions
- block unsupported claims when no evidence was collected

## Failure Pattern: Retrieval Makes the Answer Worse

### Symptom

Adding retrieved documentation degrades the answer instead of improving it.

### Likely Causes

- low-relevance retrieval results
- duplicated or contradictory context
- too much retrieved text relative to the core request
- stale documentation

### What to Check

1. Inspect the exact retrieved chunks.
2. Check whether the top-ranked chunks actually answer the user's question.
3. Check whether retrieved text displaced more relevant conversation state.
4. Check timestamps and source quality.

### Typical Fixes

- improve retrieval ranking
- reduce chunk size
- deduplicate retrieved context
- add source filtering or freshness constraints

## Runtime vs Model Diagnosis

A useful rule of thumb:

- if the right fact was never in the payload, it is a runtime or retrieval problem
- if the wrong tool was run, it is usually a runtime, schema, or instruction problem
- if the right evidence was present and the answer still failed, it is more likely a model reasoning problem

This is not perfect, but it prevents blaming the model for upstream packaging bugs.

## Minimal Observability Standard

If you are building one of these systems, log enough to reconstruct a turn after the fact:

- request id
- session id
- model id
- prompt token count
- output token count
- stop reason
- tool calls and results
- time to first token
- total turn latency
- truncation or summarization events

If those fields are missing, debugging quality drops sharply.

## Practical Workflow

When a bad answer is reported:

1. reconstruct the exact turn
2. verify whether the missing fact was present in the payload
3. verify whether the tool loop executed correctly
4. verify whether truncation or retrieval degraded the context
5. only then classify the issue as model reasoning quality

This workflow is slower than guessing, but faster than repeatedly fixing the wrong layer.
