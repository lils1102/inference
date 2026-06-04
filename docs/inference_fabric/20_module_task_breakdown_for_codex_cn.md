# 20. Module Task Breakdown for Codex

## 1. 本文件结论

后续 Codex 任务必须从文档、ADR 和 benchmark plan 派生，不应直接进入 C++/CUDA/RDMA/scheduler/runtime/kernel 实现。每个实现任务前必须有模块边界、指标、MVP 和风险。

## 2. 模块目标

把 Phase 1 拆成可派发的研究、设计、benchmark 和实现前置任务，保持 Phase 1/Phase 2 边界清晰。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

任务按 Documentation -> Benchmark Harness -> Runtime Skeleton -> Module MVP -> Optimization RFC -> Kernel RFC 分层。每个任务必须包含输入文档、禁止项、验收指标、baseline、rollback。Phase 2 KVStore 相关任务只能是 seam 或未来 ADR，不得进入 Phase 1 implementation backlog。

## 6. 数据结构草案

CodexTask(id, module, phase, inputs, outputs, metrics, forbidden_scope, reviewer)；Milestone(name, tasks, exit_criteria)；ImplementationTicket(module, mvp, benchmark, risks)。

## 7. 关键 API 草案

derive_tasks_from_doc(module)、validate_task_scope(task)、attach_benchmark(task, suite)、mark_phase2_seam(task)。

## 8. 执行流程

先完成 benchmark discovery 和 baseline truth layer，再做 execution graph/serving skeleton，再接 memory/KV/MoE/scheduler/speculative/Agent/structured/kernel。每个 optimization 先写 RFC 和 benchmark，不直接默认实现。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

任务清单、模块依赖图、阶段门、禁止项检查表和后续 PR 切分建议。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
