# ADR-0002: General Serving Is the Entry Ticket

Status: Accepted

## Context

Inference Fabric aims to become a next-generation distributed-native general inference engine, not an Agent-only runtime. The current SOTA ecosystem already provides broad general-serving capabilities: OpenAI-compatible APIs, continuous batching, prefix caching, structured output, speculative decoding, quantization, LoRA, and multi-node deployment.

## Decision

General serving capability is mandatory for Phase 1. Agent-aware capability is a differentiator, not a substitute for general serving.

Phase 1 must design for:

- OpenAI-compatible API.
- Streaming generation.
- Chat / completion serving.
- Structured output and tool calling.
- HF safetensors loader.
- Continuous batching.
- Chunked prefill.
- Paged KV baseline.
- Prefix cache baseline.
- Speculative decoding baseline.
- Multi-node serving baseline.
- Observability / metrics / profiling.

## Consequences

- The engine must not be perceived as a narrow Coding Agent serving system.
- Ordinary LLM serving must not be a second-class path.
- Agent-aware optimizations must be optional and additive.

## Alternatives Considered

- Agent-only design: rejected because it blocks general adoption.
- DeepSeek-only design: rejected because it prevents becoming a general inference engine.
- vLLM-compatible wrapper: rejected because it does not create a native distributed architecture.

## Implementation Notes

Codex should document general-serving modules separately from Agent Metadata Runtime. The architecture should route normal requests through a General Serving Context and only activate Agent Context when metadata/hints are provided.