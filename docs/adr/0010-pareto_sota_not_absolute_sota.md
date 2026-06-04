# ADR-0010: Target Pareto SOTA, Not Absolute SOTA on Every Metric

Status: Accepted

## Context

推理指标之间存在冲突：最低 TTFT、最低 TPOT、最高 throughput、最低 p99、最高 GPU utilization、最低 HBM usage、最强隔离、最广模型兼容无法在所有模型和 workload 上同时绝对第一。

## Decision

目标是 Target Pareto SOTA：在目标 benchmark suite 上，在同等硬件、模型、精度、workload、SLO 下，没有当前组合式 SOTA 推理栈能全面支配 Inference Fabric。目标 workload 包括 MoE serving、long-context、high-prefix-reuse、Agent/Coding Agent workflow、distributed multi-node serving、draft/verify serving。

## Consequences

项目不得宣称所有场景绝对第一，也不得写永远快于 vLLM/SGLang。所有优势必须绑定 workload、配置、指标和实测证据。

## Alternatives Considered

absolute SOTA on every metric：物理和工程上不现实。只优化吞吐：忽略 SLO 和 Agent task。只优化 Agent：丢失通用 serving 入场券。

## Implementation Notes

PR、文档和后续 benchmark 报告必须写清楚 baseline、硬件、模型、精度、workload、SLO 和未确认信息。
