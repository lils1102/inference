# Inference Fabric 文档入口

## 1. 本文件结论

本目录是 Inference Fabric Phase 1 的架构事实层、模块设计、benchmark 计划和 Codex handoff 入口。Phase 1 只做 distributed memory-native GPU inference engine，Phase 2 仅保留 persistent KVStore seam。

## 2. 模块目标

把项目边界、模块顺序、ADR 约束、benchmark-first 原则和后续任务拆解放在同一入口，避免后续实现把 storage、3FS、CXL/DPU 或竞品 backend 混入 Phase 1。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

文档树按从约束到模块再到 benchmark 的顺序组织：00-02 定义项目和性能模型，03-05 建立 SOTA/硬件/竞品事实层，06-15 设计执行图、serving、内存、KV、MoE、scheduler、speculative、Agent metadata、kernel、structured output，16-20 给出 benchmark、Phase 2 seam、风险、生态和任务拆解。

## 6. 数据结构草案

DocumentIndex(path, owner_module, phase, status, blocking_adr, benchmark_mapping)；ADRIndex(id, status, decision, phase_constraint)；PhaseBoundary(name, includes, excludes, seam_only_items)。

## 7. 关键 API 草案

read_doc(path)、validate_phase_boundary(doc)、list_required_metrics(module)、derive_codex_tasks(module)。这些是文档治理 API 草案，不是运行时代码接口。

## 8. 执行流程

先读 README 与 ADR，再读 00-05 的事实层，随后按 06-15 进入模块设计，最后用 16 的 benchmark plan 和 20 的任务拆解派生实现任务。任何实现前必须回看 01、04、17 和 ADR-0004/0005/0006。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

完整 Markdown 文档树、ADR、benchmark plan、Phase 1/Phase 2 边界和可派生任务列表。

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
