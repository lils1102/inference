# ADR-0001: Next-generation Distributed Inference Engine

Status: Accepted

## Context

Inference 的目标不是做 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake 或 3FS 的封装，而是构建面向 Agent/workflow、MoE、long-context、high-prefix-reuse、multi-node serving、draft/verify 和 task-level cost optimization 的通用 GPU 推理引擎。

## Decision

项目定位为 Inference Fabric：Agent-aware、MoE-native、KV-memory-native、Distributed-native 的通用 GPU 推理引擎。系统中心是 Distributed Execution Graph。竞品只能作为 benchmark 对照、技术参考和竞争对象，不作为 execution backend。

## Consequences

通用 serving 成为 Phase 1 入场券；Distributed Scheduler、Distributed Memory Fabric、In-memory KV/State Runtime、MoE Runtime、Speculative Runtime 和 Agent Metadata Runtime 成为核心差异化。

## Alternatives Considered

vLLM wrapper：差异化不足。Agent-only engine：通用性不足。先做 KVStore/storage：偏离 Phase 1。

## Implementation Notes

后续文档和任务必须先映射到 TTFT、TPOT、goodput under SLO、p99、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、accepted tokens per verify、GPU seconds/task 和 cost/task。
