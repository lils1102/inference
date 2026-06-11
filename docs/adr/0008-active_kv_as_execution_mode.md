# ADR-0008: Active KV as Execution Mode

Status: Accepted

## Context

“Active KV”容易被误解为存储层、特殊内存硬件或 offload 设备参与计算。Phase 1 没有 CXL、DPU、SmartNIC offload、PIM、FPGA，也没有 KVStore 内部设计。

## Decision

Phase 1 中更准确的名称是 KV-owner-side attention execution 或 KV-resident attention execution mode。执行者是持有 hot KV 的 GPU group。Scheduler 可在 local attention、KV migration then local attention、KV-owner-side attention、remote-prefix + local-suffix attention、prefix recomputation 之间选择。

## Consequences

Active KV 只是 execution mode，不是存储层，不是 mandatory path。它适用于 shared-prefix/high-prefix-reuse 场景，但收益不确定，需要 benchmark 确认。

## Alternatives Considered

把 Active KV 解释为存储层能力：否决。把它作为所有 remote KV hit 的固定路径：否决。完全删除 owner-side attention：否决，因为 shared-prefix 场景可能有收益。

## Implementation Notes

文档应优先使用 KV-owner-side attention execution。不得把持久化存储、外部存储组件或 offload 设备写成 attention 执行者。
