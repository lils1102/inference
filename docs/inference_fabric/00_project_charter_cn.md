# 00. Project Charter：Inference Fabric

> Productization update: 当前开发目标已升级为一次性交付完整产品化首版。本文保留原始项目 charter 和风险边界；实际工程开发以 `21_productization_full_scope_cn.md` 到 `30_engineering_execution_backlog_cn.md` 以及 ADR-0011 为主。

## 1. 本文件结论

Inference Fabric 是 Agent-aware、MoE-native、KV-memory-native、Distributed-native 的通用 GPU 推理引擎。它必须先具备通用 serving 入场券，再在 Agent/workflow、MoE、long-context、高 prefix reuse、多节点和 speculative runtime 上形成 Target Pareto SOTA。

## 2. 模块目标

给出项目定位、目标 workload、硬件目标、Phase 1 交付边界、竞品关系和成功标准，作为所有后续模块文档的上位约束。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

系统中心是 Distributed Execution Graph。API Compatibility Layer 接收 OpenAI-compatible 请求，经 Request Normalizer / Metadata Extractor 生成 General Serving Context 和可选 Agent Context，再交给 Global Distributed Scheduler，最后映射到 Model Runtime、KV Memory Runtime、MoE Runtime、Speculative Runtime、Communication Fabric 和 GPU Kernel Runtime。

## 6. 数据结构草案

ProjectCharter(project_name, target_workloads, phase1_includes, phase1_excludes, target_hardware, success_metrics)；WorkloadProfile(type, model, prompt_len, output_len, reuse_pattern, slo)；SotaClaim(scope, baseline, metric_set, evidence)。

## 7. 关键 API 草案

define_workload_suite()、register_baseline_stack(name, version, config)、record_sota_claim(metric_set, evidence_uri)、reject_out_of_scope_design(reason)。

## 8. 执行流程

项目从 benchmark-first 文档开始：确定目标 workload，列出基线栈，实测 32×H20 拓扑，建立性能模型，再进入模块 MVP。任何跨 Phase 边界的设计必须先进入 ADR 或 Phase 2 seam 文档。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

项目 charter、Phase 1 范围、成功指标、竞品边界和硬约束清单。

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
