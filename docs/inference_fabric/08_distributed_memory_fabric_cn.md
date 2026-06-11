# 08. Distributed Memory Fabric

## 1. 本文件结论

Distributed Memory Fabric 是 Phase 1 的显式运行时内存层，不是 CXL 式透明远端内存，也不是 NVMe SSD 存储层。它管理 L0 local HBM、L1 same-node peer HBM、L2 cross-node GPU HBM 和 L3 CPU DRAM/pinned staging。

## 2. 模块目标

为 hot KV、activation、expert dispatch、KV migration、compaction、pin/unpin/eviction 和 KV-owner-side attention placement 提供统一内存抽象与 cost model。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Memory Fabric 暴露 block ownership、placement、lifetime、pinning、movement 和 pressure。所有跨 GPU/节点访问都是显式操作：copy、migrate、materialize、dispatch 或 owner-side execution。CPU DRAM/pinned memory 只用于 metadata、staging 和 warm buffers，不作为每 token decode KV 来源。

## 6. 数据结构草案

MemoryBlock(block_id, kind, bytes, owner, tier, ref_count, pin_state, last_used)；PlacementMap(block_id, replicas, owner_gpu_group, freshness)；MemoryPressure(gpu_id, free_hbm, fragmentation, eviction_candidates)；TransferPlan(src, dst, bytes, transport, estimated_ms)。

## 7. 关键 API 草案

allocate_block(kind, bytes, placement_hint)、move_block(block_id, dst)、pin_block(block_id, ttl)、evict_block(block_id)、compact_gpu(gpu_id)、estimate_transfer(src, dst, bytes)。

## 8. 执行流程

KV append 分配 L0 blocks；prefix reuse 命中后检查 owner；scheduler 在 local attention、迁移后本地 attention、KV-owner-side attention、remote-prefix + local-suffix merge、prefix recompute 中选择；memory fabric 执行 transfer 并上报 bytes/latency。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

HBM block allocator、block table、placement map、显式 GPU-GPU transfer、pin/evict/compact、transfer metrics 和 HBM pressure feedback。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
