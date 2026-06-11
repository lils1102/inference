# ADR-0005: No CXL, No DPU, No 3FS Dependency

Status: Accepted

## Context

目标环境是 GPU、CPU host、NCCL/RDMA/GDR/P2P、CPU DRAM/pinned memory 的通用集群。CXL、DPU/SmartNIC offload、PIM、FPGA 不是 Phase 1 依赖。3FS 是竞品组合栈组件，不是内部依赖。

## Decision

Phase 1 明确 No CXL、No DPU / SmartNIC offload、No PIM、No FPGA、No 3FS dependency。跨节点状态访问必须显式建模和 benchmark。

## Consequences

Scheduler 必须对 KV movement、activation dispatch、expert dispatch、communication latency 和 HBM pressure 建 cost model，不能依赖不可用硬件或外部存储系统。

## Alternatives Considered

依赖 3FS 做内部存储：否决。依赖 DPU/CXL 做远端状态：否决。把 RDMA/GDR 视为智能计算层：否决。

## Implementation Notes

3FS 只能在 competitor stack analysis 和 benchmark plan 中出现，例如 Dynamo + vLLM/SGLang + LMCache + Mooncake + 3FS。
