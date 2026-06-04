# ADR-0007: MoE Expert as a System Resource

Status: Accepted

## Context

MoE 推理受 expert placement、all-to-all dispatch/combine、expert queue、hotness skew、grouped GEMM efficiency 和跨节点通信影响。把 expert 当作局部 operator 会隐藏系统瓶颈。

## Decision

Inference Fabric 把 MoE expert 建模为系统级资源。Scheduler 必须能读取 expert_id、layer_id、placement、replica、queue depth、expected service time、communication cost、HBM residency 和 routing locality。

## Consequences

MoE runtime 成为核心模块，expert locality 可以与 KV locality、topology 和 SLO 联合优化。

## Alternatives Considered

仅依赖框架默认 MoE kernel：不足。静态 expert placement 永久不变：不足。把所有 expert 跨节点均匀摊开且不 benchmark：风险过高。

## Implementation Notes

默认并行策略参考 ADR-0009；expert dispatch latency、expert imbalance 和 GPU seconds/task 是必测指标。
