# 11. Distributed Scheduler

## 1. 本文件结论

Scheduler 不能一开始承诺全局最优，必须分阶段演进：V0 continuous batching + chunked prefill + TP/EP/DP placement；V1 KV locality；V2 MoE expert locality；V3 draft/verify；V4 Agent metadata；V5 learned cost model / auto-tuning。

## 2. 模块目标

在 SLO、GPU utilization、HBM pressure、KV locality、expert locality、communication cost 和 Agent task cost 之间做可解释、可观测、可回退的调度。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Scheduler 输入包括 ExecutionGraph、TopologyProfile、KVIndex、MemoryPressure、ExpertQueue、DraftVerifyState、AgentContext 和 SLO。输出 GraphPlan，包括 batch、placement、parallelism、KV action、expert dispatch、draft/verify plan 和 fallback。

## 6. 数据结构草案

ScheduleState(queues, gpu_load, hbm_pressure, kv_index, expert_queues, slo)；CandidatePlan(actions, estimated_cost, risk, fallback)；AdmissionDecision(accept, queue, reject_reason)；SchedulerVersion(V0..V5, enabled_features)。

## 7. 关键 API 草案

admit_request(context)、build_candidate_plans(graph)、score_plan(plan, cost_model)、commit_plan(plan)、preempt_or_rebatch(reason)、update_cost_model(trace)。

## 8. 执行流程

每个 tick 读取等待队列和资源状态，生成候选 batches 与 placements，按 SLO/goodput/cost 排序，提交 plan。执行后收集 TTFT、TPOT、p99、KV bytes、expert latency、accepted tokens、GPU seconds/task 回灌。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

V0/V1：continuous batching、chunked prefill、basic TP/EP/DP placement、KV locality scoring、HBM pressure eviction feedback 和可解释调度日志。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
