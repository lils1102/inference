# ADR-0004: Phase 1 Is Distributed Memory Only, No KVStore

Status: Accepted

## Context

持久化全闪 KVCache Store 对未来 cold restore 和 durable reuse 有价值，但会把 Phase 1 从推理热路径扩大到存储系统，且容易混淆 runtime memory 与 storage。

## Decision

Phase 1 只做 distributed memory-native GPU inference：L0 local HBM、L1 same-node peer HBM、L2 cross-node GPU HBM through RDMA / GPUDirect RDMA / NCCL / P2P、L3 CPU DRAM / pinned metadata/staging。

## Consequences

所有 KV placement、migration、compaction、pin、eviction 和 owner-side attention 都是运行时内存动作；persistent KVStore、NVMe restore、GDR storage I/O、restart recovery、storage-backed Agent State 均排除。

## Alternatives Considered

把 KVStore 放入 Phase 1：否决。把 NVMe 作为 decode hot KV 来源：否决。只做单机 HBM：否决，因为目标是 distributed-native。

## Implementation Notes

Hot decode KV 必须在 GPU HBM。任何 storage 相关内容只能出现在 `17_phase2_persistent_kvstore_seam_cn.md`。
