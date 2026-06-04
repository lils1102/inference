# 10. MoE-native Runtime

## 1. 本文件结论

MoE expert 是系统级资源，不是普通 FFN kernel 细节。Phase 1 必须把 expert placement、queueing、hotness、replication、dispatch/combine、grouped GEMM 和 expert imbalance 纳入 scheduler 与 benchmark。

## 2. 模块目标

降低 MoE serving 中的 expert dispatch latency、all-to-all overhead、small GEMM inefficiency、hot expert p99 queueing 和跨节点通信放大。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

MoE runtime 分离 Router、ExpertPlacementTable、DispatchPlanner、ExpertQueue、GroupedGemmBatcher 和 CombinePlanner。默认倾向 intra-node TP 与 cross-node EP/DP/P-D/state sharding，但所有 parallelism 策略都必须由 benchmark discovery 和模型 workload 确认。

## 6. 数据结构草案

ExpertPlacement(model, layer, expert_id, replicas, gpu_group, load)；ExpertRoute(token_batch, topk_experts, scores)；ExpertQueue(expert_id, depth, p50_ms, p99_ms)；DispatchBatch(src, dst_expert, tokens, bytes, priority)。

## 7. 关键 API 草案

route_tokens(hidden_states)、plan_dispatch(routes, topology)、enqueue_expert(batch)、run_grouped_gemm(expert_batches)、combine_outputs(dispatch_id)、update_expert_hotness(stats)。

## 8. 执行流程

每个 MoE layer 先 router top-k，再按 ExpertPlacementTable 聚合 tokens，dispatch 到 expert owners，expert queue 合批执行 grouped GEMM，combine 返回 hidden states。scheduler 根据 hotness 和 imbalance 调整 placement/replication 候选。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

静态 expert placement、basic all-to-all dispatch/combine、expert queue metrics、grouped GEMM baseline、hot expert profiling 和 cross-request coalescing 设计。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
