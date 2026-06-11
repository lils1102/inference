# ADR-0002: General Serving Is the Entry Ticket

Status: Accepted

## Context

主流推理栈已经提供 OpenAI-compatible API、streaming、continuous batching、chunked prefill、paged KV、prefix cache、structured output、tool calling、speculative decoding、quantization、LoRA、多节点部署和 observability。没有这些能力，Inference Fabric 无法成为通用引擎。

## Decision

Phase 1 必须把 general serving 作为入场券。Agent-aware、MoE-native 和 KV-memory-native 是增强层，不替代普通 serving。

## Consequences

项目必须优先补齐协议、模型加载、batching、KV baseline、structured output 和 metrics；否则后续差异化无法被用户采用或 benchmark。

## Alternatives Considered

只做 Agent runtime：否决。只做内核库：否决。只做分布式调度层并复用竞品 engine：否决。

## Implementation Notes

通用 serving MVP 的每个子能力都需要独立 benchmark 或兼容性测试，并进入 `07_general_serving_entry_ticket_cn.md`。
