# ADR-0008: Active KV as Execution Mode

Status: Accepted

## Context

Earlier designs used the term Active KV. Without CXL, DPU, SmartNIC offload, or Phase-1 KVStore, this term can be misunderstood as storage-side or memory-fabric-side computation.

## Decision

Active KV is defined as an execution mode:

```text
KV-owner-side attention execution
```

If a GPU group owns hot KV, the scheduler may send query/hidden state to that GPU group and execute shared-prefix or remote-prefix attention there, instead of moving the entire KV block to the requesting GPU.

This is a scheduler option, not a mandatory path.

## Consequences

Scheduler can choose among:

- local attention
- migrate KV then local attention
- KV-owner-side attention
- prefix recomputation
- keep current placement

The executor for Active KV is a GPU group that owns hot KV, not a KVStore, DPU, CXL memory node, or storage device.

## Alternatives Considered

- Treat Active KV as storage-side compute: rejected.
- Treat Active KV as mandatory for all remote KV hits: rejected.
- Ignore remote-prefix execution: rejected because high-prefix-reuse and shared-prefix workloads may benefit.

## Implementation Notes

Codex should prefer the precise term `KV-owner-side attention execution` in Phase 1 docs, and may mention `Active KV` as a shorthand. It must not describe KVStore or NVMe SSD as computing attention.