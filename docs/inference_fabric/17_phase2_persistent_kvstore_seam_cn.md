# 17. Phase 2 Persistent KVStore Seam

> Productization update: 当前产品化首版已经把 persistent KV/state storage 纳入完整产品范围。本文保留“storage 不得进入每 token decode hot path”的风险边界；具体产品实现契约以 `26_memory_kv_state_storage_spec_cn.md` 为准。

## 1. 本文件结论

Persistent all-flash KVCache Store 只属于 Phase 2 seam。它是未来基于 NVMe SSD 的持久化存储层，不是 Phase 1 内存层，不在 decode hot path，不作为 KV-owner-side attention 执行者。

## 2. 模块目标

定义 Phase 1 与未来 persistent KVStore 的接口边界，只保留 seam，不做内部存储设计。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。
- 不在 Phase 1 内部设计 NVMe layout、flash index、failure recovery 或 storage-backed Agent State。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Phase 1 只允许定义 SnapshotExport、SnapshotImport、PrewarmHint、DurableStateRef 等抽象边界。不得设计全闪索引、NVMe layout、GDR storage I/O、failure recovery 协议或 storage-backed Agent State。未来 Phase 2 可用于 KV snapshot、long-context restore、P/D handoff、cross-session durable reuse、failure recovery、offline prewarm、workflow replay。

## 6. 数据结构草案

DurableStateRef(id, model, token_span, metadata, created_at)；SnapshotManifest(block_refs, dtype, layout, checksum)；PrewarmHint(state_ref, target_gpu_group, deadline)。这些是 seam 级描述，不代表 Phase 1 实现。

## 7. 关键 API 草案

export_snapshot_seam(state_ref)、import_snapshot_seam(durable_ref)、request_prewarm_seam(hint)、report_restore_cost_seam(metrics)。Phase 1 只能 stub 或文档化这些接口。

## 8. 执行流程

Phase 1 runtime 可以在结束时产生“可导出状态”的 metadata，但不落盘、不恢复、不在热路径读取。Phase 2 若实现，必须先通过新的 ADR 和 benchmark plan，证明不会污染 hot decode path。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

接口 seam、禁止项、未来用例、需要新增 ADR 的条件和 benchmark 验收要求。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
