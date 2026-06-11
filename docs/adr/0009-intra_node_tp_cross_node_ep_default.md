# ADR-0009: Intra-node TP and Cross-node EP as Default

Status: Proposed

## Context

目标硬件为 4 nodes × 8 × H20 GPUs，但 NVLink/NVSwitch/PCIe/RDMA/GDR 拓扑不确定，需要 benchmark discovery 确认。跨节点 TP 可能带来高频 collective，同节点 TP 通常更容易控制延迟，但不能静态假设。

## Decision

默认 planning assumption 是 intra-node Tensor Parallelism；cross-node 使用 Expert Parallelism / Data Parallelism / Prefill-Decode split / state sharding。跨节点 TP 不禁止，但只有 benchmark 证明某模型、硬件拓扑、workload 下有利时才启用。

## Consequences

Scheduler 和 model runtime 先围绕 node-local GPU group 设计，同时保留 cross-node TP、PP、DP、P/D disaggregation 的实验入口。

## Alternatives Considered

cross-node TP by default：风险高。single-node-only：不满足目标。固定 parallelism 不 benchmark：否决。

## Implementation Notes

Benchmark 必须比较 intra-node TP + cross-node EP、cross-node TP、cross-node PP、cross-node DP、P/D disaggregation，并记录 NCCL allreduce/alltoall、RDMA/GDR、P2P、HBM 和 p99。
