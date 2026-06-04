# ADR-0006: KVStore Is Future Persistent Storage, Not Memory

Status: Accepted

## Context

The project owner clarified that the distributed all-flash KVStore is NVMe SSD-based persistent storage. It is not memory, not remote memory, and not a decode hot-path memory tier.

## Decision

The native distributed all-flash KVCache Store is a Phase 2 persistent storage layer. It is not part of Phase 1 runtime memory.

It may later support:

- KV snapshot persistence.
- Long-context KV restore.
- P/D KV handoff.
- Cross-session durable reuse.
- Failure recovery.
- Offline prewarm.

It must not be described as:

- CXL-like memory.
- Remote memory.
- Distributed memory tier.
- Per-token decode KV source.

## Consequences

- Phase 1 focuses on GPU HBM / peer HBM / CPU DRAM runtime memory.
- Hot decode KV must live in GPU HBM.
- Future GDR KV interface is storage I/O optimization, not load/store memory semantics.

## Alternatives Considered

- Treat all-flash KVStore as memory tier: rejected.
- Include KVStore in Phase 1: rejected.

## Implementation Notes

Codex must keep KVStore content in Phase 2 seam documents only. It must not design KVStore internals or place it in Phase 1 hot path.