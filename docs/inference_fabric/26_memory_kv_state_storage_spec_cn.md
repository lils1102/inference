# 26. Memory, KV, State, and Storage Spec

## 1. 本文件结论

产品化首版包含两个状态系统：hot runtime memory system 和 durable persistent state store。前者服务 GPU decode hot path，后者服务 snapshot、restore、prewarm、failure recovery 和 cross-session durable reuse。二者必须通过明确边界连接，不允许 storage 成为每 token decode KV 来源。

## 2. Runtime Memory Tiers

- L0：local GPU HBM。
- L1：same-node peer GPU HBM。
- L2：cross-node GPU HBM via explicit RDMA/GDR/NCCL/P2P transfer。
- L3：CPU DRAM / pinned memory / metadata / staging。

操作：

- allocate
- append
- move
- materialize
- pin
- unpin
- evict
- compact
- owner-side attention

## 3. Persistent State Store

职责：

- state snapshot
- manifest catalog
- checksums
- TTL / retention
- async restore
- prewarm into HBM
- crash recovery
- offline warmup

非职责：

- per-token decode KV read
- transparent remote memory
- storage-side attention
- 3FS dependency

## 4. Core Structures

`KVBlock`：

```text
block_id
model_id
layer_id
token_start
token_end
head_range
dtype
layout
owner_gpu_group
bytes
checksum
```

`PrefixEntry`：

```text
prefix_hash
token_count
block_ids
owner
hit_count
last_used
tenant_scope
```

`PersistentStateRef`：

```text
state_id
manifest_version
model_id
token_span
snapshot_uri
checksum
created_at
ttl
metadata
```

## 5. APIs

Runtime：

```text
AllocateKVBlock(req) -> KVBlockHandle
AppendKV(req) -> AppendResult
LookupPrefix(req) -> PrefixResult
PlanKVAction(req) -> KVActionPlan
MoveKV(req) -> TransferResult
CompactHBM(req) -> CompactResult
```

Storage：

```text
SnapshotState(req) -> PersistentStateRef
RestoreState(req) -> RestoreJob
PrewarmState(req) -> PrewarmJob
GetManifest(state_id) -> SnapshotManifest
DeleteState(state_id) -> DeleteResult
VerifySnapshot(state_id) -> VerifyResult
```

## 6. Lifecycle

Request lifecycle：

1. prefix lookup
2. optional persistent restore/prewarm before prefill/decode
3. HBM materialization
4. decode hot path uses HBM only
5. optional snapshot after completion or interval
6. eviction based on policy

## 7. Storage Manifest

Manifest 必须包含：

- model identity
- tokenizer identity
- dtype/layout
- block list
- checksum
- metadata scope
- compatibility version
- encryption status
- retention policy

## 8. Correctness

必须保证：

- block checksum verification。
- model/tokenizer mismatch rejection。
- snapshot restore idempotency。
- partial snapshot cleanup。
- COW branch consistency。
- tenant isolation。

## 9. Metrics

- HBM free bytes
- HBM fragmentation
- KV bytes moved
- prefix hit rate
- snapshot bytes written
- restore bytes read
- prewarm latency
- restore success rate
- eviction count
- checksum failure count

## 10. Tests

- allocator unit tests。
- prefix cache golden。
- COW branch tests。
- snapshot/restore roundtrip。
- crash during snapshot。
- manifest compatibility。
- no storage read in decode loop invariant。

## 11. Implementation Notes

- persistent store 可以先用 local filesystem backend 实现 contract，再替换为 distributed NVMe backend。
- storage backend 必须是可插拔接口，但 3FS 只能作为外部 benchmark 对照，不得作为默认依赖。
