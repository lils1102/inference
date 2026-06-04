# ADR-0007: MoE Expert as a System Resource

Status: Accepted

## Context

Modern MoE inference is constrained by expert placement, expert dispatch, all-to-all communication, grouped GEMM efficiency, expert imbalance, and hot expert skew. Treating MoE as a local model operator is insufficient for a native distributed inference engine.

## Decision

Inference Fabric treats MoE experts as first-class distributed system resources.

The scheduler must eventually reason about:

- expert_id / layer_id / model_id
- node_id / gpu_id
- expert hotness
- queue depth
- replica count
- expected service time
- communication cost
- HBM residency
- routing locality

## Consequences

- Expert placement becomes part of the global scheduler.
- Expert locality can be jointly optimized with KV locality and topology.
- MoE runtime becomes a core differentiator instead of an implementation detail.

## Alternatives Considered

- Treat expert parallelism as a backend config only: rejected because it does not expose enough control for Agent/MoE/native distributed scheduling.
- Rely only on generic all-to-all collectives: rejected because expert-aware dispatch, batching, and replication require semantic scheduling.

## Implementation Notes

Codex should design an `ExpertPlacementTable` and MoE Runtime document. It should compare vLLM/SGLang expert parallel capabilities but must frame our advantage as system-level expert scheduling, not merely EP support.