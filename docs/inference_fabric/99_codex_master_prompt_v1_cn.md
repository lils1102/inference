# 99. Codex Master Prompt v1

> Productization update: 后续 Codex 若要开发完整产品，必须优先读取 `21_productization_full_scope_cn.md` 到 `30_engineering_execution_backlog_cn.md` 和 ADR-0011。本文原先的 Phase 1 口径仅作为历史范围控制和风险边界。

## 1. 本文件结论

本文件是给后续 Codex 的主提示词和硬约束汇总。当前产品化开发口径要求一次性交付完整 Inference Fabric 产品：通用 serving、分布式执行图、调度、内存/KV、persistent state storage、MoE、speculative、Agent metadata、structured generation、kernel runtime、benchmark、部署、运维、安全和发布质量门槛。

## 2. 模块目标

让后续 Codex 在上下文不足时仍能恢复正确项目定义、产品化范围、硬件不确定性和 benchmark-first 工作方式。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- 产品化首版包含自研 persistent KV/state storage，但不得把 storage 放入 per-token decode hot path，也不得依赖 3FS。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

提示词包含角色、项目定义、硬件目标、Phase 1 范围、硬约束、分布式内存层、系统中心、文档清单、ADR 清单、竞品比较、GPU kernel 策略、scheduler 阶段、benchmark plan 和输出要求。

## 6. 数据结构草案

PromptSection(id, purpose, hard_constraints, required_outputs)；ForbiddenScope(item, phase, reason)；HandoffChecklist(doc_tree, adr, benchmark, pr_description)。

## 7. 关键 API 草案

load_master_prompt()、validate_codex_output(files)、check_forbidden_scope(diff)、summarize_pr_requirements()。这些是 handoff 流程草案，不是运行时代码。

## 8. 执行流程

后续 Codex 先读 README、ADR-0011、21-30，再按 30 的 backlog 执行具体任务。若发现需求涉及 CXL/DPU、3FS dependency 或 storage hot decode path，应停止并提出 ADR 讨论。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

完整主提示词、产品化文件清单、禁止项、benchmark 指标和 PR 描述要求。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解；具体代码开发任务必须从 21-30 的产品化规格派生。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；persistent KV/state storage 只能用于 snapshot、restore、prewarm、recovery 和 durable reuse；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。

## 附录：既有边界摘要

本次 Full Documentation Pass 保留 main 分支既有边界，并将其结构化到上方 13 个章节：

- 项目不是 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake 或 3FS 的 wrapper/backend。
- Phase 1 只做 distributed memory-native GPU inference engine，不设计 persistent KVStore、NVMe restore、GDR storage I/O、storage-backed Agent State 或 3FS 替代品。
- Distributed Memory Fabric 是显式运行时内存层，覆盖 L0 local HBM、L1 peer HBM、L2 cross-node GPU HBM 和 L3 CPU DRAM/pinned metadata/staging。
- 系统中心是 Distributed Execution Graph；Agent 能力是可选 metadata layer，不强迫外部 Agent app 改协议。
- 所有拓扑、通信、kernel 和竞品性能判断都不确定，需要 benchmark discovery 或调研确认。
