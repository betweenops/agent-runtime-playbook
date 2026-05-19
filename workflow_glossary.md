# Glossary

One-line definitions to read alongside [workflow_overview.md](./workflow_overview.md).

## Agentic CLI

The local client that captures your prompt, assembles context, runs tools, streams output, and persists session state. Codex, Claude Code, Open Code, Cursor, and Copilot Chat are all examples.

## API Provider

The hosted service that authenticates requests, routes them to a model, applies quotas, and streams responses back.

## LLM

A large language model — a neural network that predicts the next token given the tokens so far.

## Token

A model-readable chunk of text. Smaller than a sentence, not always a word.

## Tokenization

Splitting text into tokens the model can consume.

## Context Window

The full set of tokens the model can consider in one request. Usually includes system + developer instructions, chat history, the latest user message, and any tool output.

## Inference

The live execution of a trained model to produce output for a request. No weights are updated.

## Training

The offline process that adjusts model weights before deployment. Not happening during your request.

## Embeddings

Vector representations of tokens. The model operates on vectors, not raw text.

## Transformer

The neural-network architecture most modern LLMs use. Stacked attention layers process token relationships.

## Decoding

Picking the next token from the model's probability distribution, appending it, and repeating.

## Tool Call

A structured action the model emits, asking the CLI to do something (read a file, run a shell command, hit an API). The model does not execute it itself.

## Session State

Locally stored history and metadata the CLI uses to reconstruct context across turns.

## Stateless Model

The model has no memory across requests. Any apparent memory is recreated by re-sending context.

## Streaming

Returning generated tokens incrementally instead of waiting for the full response.

## Cached Tokens

Tokens from a previously processed prompt prefix that the provider reused for performance or cost.

## Reasoning Tokens

A provider-specific category representing additional internal inference budget used during more complex reasoning.
