# Workflow Glossary

This glossary is a quick-reference companion to [workflow_overview.md](./workflow_overview.md).

## Codex

The local client or runtime that manages prompts, context, tools, streaming output, and session state.

## API Provider

The hosted service that authenticates requests, routes them to a model, applies quotas, and returns responses.

## LLM

A large language model that predicts the next token based on the tokens already in context.

## Token

A model-readable unit of text. Tokens are smaller than sentences and are not always the same as words.

## Tokenization

The process of splitting text into tokens that the model can consume.

## Context Window

The full set of tokens the model can consider for a single request. This often includes system instructions, chat history, the latest user message, and tool output.

## Inference

The live execution of a trained model to produce output for a request.

## Training

The offline process used to adjust model weights before the model is deployed for inference.

## Embeddings

Vector representations of tokens that allow the model to process relationships between pieces of text numerically.

## Transformer

The neural network architecture commonly used by modern LLMs. It processes token relationships through stacked layers and attention mechanisms.

## Decoding

The step where the model chooses the next token from its probability distribution, then repeats that process until it stops.

## Tool Call

A runtime-mediated action requested during the workflow, such as reading a file, executing a shell command, or querying another service.

## Session State

Locally stored metadata and message history that help the runtime reconstruct prior context.

## Stateless Model

A model that does not inherently remember previous requests. Any apparent memory is recreated by sending context again.

## Streaming

Returning generated tokens incrementally instead of waiting for the full response to finish first.

## Cached Tokens

Previously processed context that a provider may reuse for performance or cost efficiency.

## Reasoning Tokens

A provider-specific category that may represent additional internal compute or hidden generation budget used during more complex reasoning.
