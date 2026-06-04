# ADR-0006: KVStore Is Future Persistent Storage, Not Memory

Status: Accepted

## Context

未来自研 distributed all-flash KVCache Store 属于 NVMe SSD-based persistent storage。它不是 memory tier，不是 Phase 1 Distributed Memory Fabric，也不是 decode 每 token 热路径。

## Decision

KVStore 是 Phase 2 seam，只可用于未来 KV snapshot、long-context restore、P/D handoff、cross-session durable reuse、failure recovery、offline prewarm 和 workflow replay。

## Consequences

Phase 1 文档只能保留 seam，不设计内部 KVStore。所有热路径调度与 attention 执行仍基于 GPU HBM 和显式通信。

## Alternatives Considered

把 KVStore 命名为 L4 memory tier：否决。把 storage restore 写进 Phase 1 TPOT 模型：否决。完全不留 seam：否决，因为未来 Phase 2 需要边界。

## Implementation Notes

若未来实现 Phase 2，必须新增 ADR、benchmark suite 和 failure/correctness 设计，且不得污染 Phase 1 hot decode path。
