# 06. Distributed Execution Graph

## 1. 本文件结论

系统中心是 Distributed Execution Graph，不是 Agent Runtime 或单机 engine。Graph 把 prefill、decode、attention、KV movement、MoE dispatch/combine、draft/verify、structured output、sampling 和 metadata-driven branch 表达为可调度节点。

## 2. 模块目标

给 scheduler 一个统一 IR，使模型运行时、KV 内存、MoE、speculative、communication 和 GPU kernel 可以被统一规划、执行和观测。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Execution Graph 由 RequestGraph、StageNode、ResourceEdge 和 StateEdge 组成。Graph 节点表达 compute、memory、communication 或 control；边表达 KV/activation/logits/metadata 依赖。Graph 必须能表达 P/D disaggregation、KV-owner-side attention execution、remote-prefix + local-suffix attention、expert parallel 和 draft/verify 子图。

## 6. 数据结构草案

ExecutionGraph(graph_id, request_ids, nodes, edges, slo, metrics)；StageNode(type, placement, resource_req, estimated_cost, observed_cost)；StateEdge(kind, bytes, owner, lifetime, consistency)；GraphPlan(version, placements, batches, fallbacks)。

## 7. 关键 API 草案

build_graph(serving_context)、annotate_state_edges(graph, kv_index)、plan_graph(graph, topology_profile)、execute_graph(plan)、trace_graph(graph_id)。

## 8. 执行流程

API 请求归一化后生成 graph；scheduler 根据 topology、KV ownership、expert queues、HBM pressure 和 SLO 做 placement；executor 按 graph plan 提交 kernels/collectives/transfers；profiling 把每个 node/edge 的实际成本回写。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

支持 chat/completion 的 prefill/decode graph、paged KV state edge、MoE dispatch edge、speculative draft/verify subgraph 和基本 trace。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
