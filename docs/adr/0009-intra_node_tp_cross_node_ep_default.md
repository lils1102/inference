# ADR-0009: Intra-node TP, Cross-node EP as Default

Status: Proposed

## Context

The primary hardware target is 4 nodes × 8 H20 GPUs. Exact NVLink/NVSwitch/PCIe/RDMA topology is unknown and must be discovered by benchmark. In distributed inference, tensor parallelism across nodes can introduce high-frequency collective communication, while expert parallelism and data/prefill-decode placement often tolerate cross-node boundaries better.

## Decision

The default Phase 1 planning assumption is:

```text
Intra-node Tensor Parallelism
Cross-node Expert Parallelism / Data Parallelism / Prefill-Decode placement / KV-state sharding
```

This is a default planning assumption, not a hard rule. Benchmark discovery may override it.

## Consequences

- The scheduler and model runtime should first optimize for intra-node GPU groups.
- Cross-node communication should be minimized on latency-sensitive per-layer dense collectives.
- MoE expert placement, P/D placement, and KV/state locality become primary cross-node scheduling levers.
- NCCL allreduce/alltoall, RDMA, GPUDirect RDMA, and GPU P2P topology must be benchmarked before finalizing parallelism plans.

## Alternatives Considered

- Cross-node TP by default: rejected as a default due to high synchronization risk.
- Single-node-only design: rejected because the project targets native multi-node serving.
- Static TP/EP configuration only: rejected because the system should evolve toward topology-aware planning.

## Implementation Notes

Codex should document this as a benchmark-driven default, not as an absolute rule. The benchmark plan must include NCCL allreduce, alltoall, P2P bandwidth, RDMA/GDR checks, and model-specific TP/EP experiments.