# 18. Risks and Counterarguments

## 1. 本文件结论

Inference Fabric 的主要风险来自范围过大、通用 serving 追赶成本、硬件拓扑不确定、MoE/Distributed KV 调度复杂、Agent metadata 收益不稳定、以及竞品快速演进。风险必须转化为 benchmark 和阶段门。

## 2. 模块目标

记录反方观点、失败模式、缓解策略和需要 human review 的判断点。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

风险按 scope、performance、hardware、correctness、ecosystem、operations 分类。每个风险绑定可观测指标和退出条件。对不能在 Phase 1 证明收益的设计，降级为 seam 或 RFC，不进入默认路径。

## 6. 数据结构草案

RiskItem(id, category, severity, owner, metric, mitigation, decision_gate)；CounterArgument(claim, concern, evidence_needed)；KillCriteria(feature, threshold, fallback)。

## 7. 关键 API 草案

register_risk(item)、link_risk_to_benchmark(risk_id, suite)、evaluate_gate(feature, metrics)、document_fallback(feature)。

## 8. 执行流程

设计阶段登记风险；MVP 前定义 kill criteria；benchmark 后评估是否推进、降级或移出 Phase 1；重大边界变化必须走 ADR。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

风险表、反方观点表、kill criteria 和 human review checklist。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
