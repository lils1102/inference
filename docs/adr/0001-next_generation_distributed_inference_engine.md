# ADR-0001: Next-generation Distributed Inference Engine

Status: Accepted

## Context

Inference 的目标不是做 vLLM、SGLang、TensorRT-LLM、Dynamo、LMCache、Mooncake 或 3FS 的封装，而是构建一个面向未来 workload 的通用 GPU 推理引擎。

目标 workload 包括普通 LLM serving、MoE、long-context、high-prefix-reuse、Agent / workflow、multi-node serving、draft/verify 和 task-level cost optimization。

## Decision

项目定位为：

```text
Inference Fabric: Agent-aware、MoE-native、KV-memory-native、Distributed-native 的通用 GPU 推理引擎。
```

系统中心是 Distributed Execution Graph，而不是单机 engine 或 Agent app。

## Consequences

- 通用 serving 能力是入场券，不能明显弱于 vLLM/SGLang。
- Agent-aware、MoE-native、Distributed KV memory 和 scheduler 是差异化。
- 竞品系统只能作为 benchmark、参考和对照，不作为 execution backend。

## Alternatives Considered

- 做 vLLM wrapper：否决，差异化不足。
- 做 Coding Agent 专用引擎：否决，通用性不足。
- 做 KVStore / storage 系统：否决，偏离推理引擎主线。

## Implementation Notes

Phase 1 先实现 distributed memory-native GPU inference engine。持久化 KVStore 留给 Phase 2 seam。