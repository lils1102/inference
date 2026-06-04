# ADR-0004: Phase 1 Is Distributed Memory Only, No KVStore

Status: Accepted

## Context

The project originally explored a persistent distributed all-flash KVCache Store. That system is important for future cold-state restore and durable KV snapshots, but it expands Phase 1 scope too much and risks confusing storage with runtime memory.

## Decision

Phase 1 focuses only on distributed memory-native GPU inference.

Phase 1 includes:

- GPU HBM hot KV.
- Peer GPU HBM.
- Cross-node GPU HBM movement via RDMA / GPUDirect RDMA / NCCL / P2P.
- CPU DRAM / pinned memory for metadata and staging.
- In-memory KV/State Runtime.
- KV placement / migration / compaction / pin / eviction.

Phase 1 excludes:

- Persistent KVStore design.
- NVMe SSD KV restore.
- GDR KV storage I/O.
- Restart-time KV recovery.
- Storage-backed Agent State.

## Consequences

- Phase 1 can stay focused on GPU hot path and distributed memory performance.
- Persistent KV restore and cold-state recovery are deferred to Phase 2.
- Benchmark plans must not claim persistent KVStore capabilities for Phase 1.

## Alternatives Considered

- Build persistent KVStore in Phase 1: rejected due to scope explosion.
- Ignore future KVStore entirely: rejected because Phase 2 should preserve an interface seam.

## Implementation Notes

Codex must document Phase 2 persistent KVStore only as a seam. It must not design KVStore internals in Phase 1 docs.