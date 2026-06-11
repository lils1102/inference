# 01. Phase 1 Scope：Distributed Memory Only

> Productization update: 本文件描述的是早期范围控制。当前要求是不区分阶段、一次开发完整产品化系统；开发任务以 `21_productization_full_scope_cn.md` 到 `30_engineering_execution_backlog_cn.md` 为准。本文仍作为 hot path、hardware 和 storage 风险边界参考。

## 1. 本文件结论

Phase 1 只做 Distributed memory-native GPU inference engine。分布式内存层覆盖 GPU HBM、peer GPU HBM、cross-node GPU HBM 和 CPU DRAM / pinned memory，不设计 persistent KVStore 或 storage-backed hot path。

## 2. 模块目标

固定 Phase 1 范围，明确包含模块、排除项、hot path 语义和 Phase 2 seam，防止把持久化存储、3FS 或硬件 offload 误写入当前架构。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Phase 1 由通用 serving、Distributed Execution Graph、Distributed Scheduler、Distributed Memory Fabric、In-memory KV/State Runtime、MoE-native Runtime、Speculative Runtime、Agent Metadata Runtime、GPU Kernel Strategy 和 Benchmark/Profiling 构成。Hot decode KV 必须在 GPU HBM，跨节点 KV 只能通过显式迁移、materialization 或 KV-owner-side attention execution 处理。

## 6. 数据结构草案

PhaseScope(include_modules, exclude_modules, seam_modules)；MemoryTier(L0/L1/L2/L3, medium, latency_class, allowed_ops)；HotPathInvariant(name, forbidden_paths, validation_metric)。

## 7. 关键 API 草案

is_phase1_allowed(component)、classify_memory_tier(buffer)、plan_kv_action(block, source, target)、assert_no_storage_hot_path(execution_plan)。

## 8. 执行流程

请求进入 serving 后由 scheduler 在 L0/L1/L2/L3 之间做显式 placement。若 hot KV 不在本地 GPU，策略只能是迁移后本地 attention、在 KV owner GPU group 上执行 attention、remote-prefix + local-suffix merge 或重算 prefix。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

Level 0/1 API、基本 continuous batching、chunked prefill、paged KV、prefix cache、基础多节点通信、MoE dispatch、speculative draft/verify 和完整观测指标。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。

## 附录：既有边界摘要

本次 Full Documentation Pass 保留 main 分支既有边界，并将其结构化到上方 13 个章节：

- 项目不是 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake 或 3FS 的 wrapper/backend。
- Phase 1 只做 distributed memory-native GPU inference engine，不设计 persistent KVStore、NVMe restore、GDR storage I/O、storage-backed Agent State 或 3FS 替代品。
- Distributed Memory Fabric 是显式运行时内存层，覆盖 L0 local HBM、L1 peer HBM、L2 cross-node GPU HBM 和 L3 CPU DRAM/pinned metadata/staging。
- 系统中心是 Distributed Execution Graph；Agent 能力是可选 metadata layer，不强迫外部 Agent app 改协议。
- 所有拓扑、通信、kernel 和竞品性能判断都不确定，需要 benchmark discovery 或调研确认。
