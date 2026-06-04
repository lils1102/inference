# 13. Agent Metadata Runtime

## 1. 本文件结论

Agent-aware 能力是可选 metadata layer，不是强制 Agent protocol。Level 0 支持普通 OpenAI API；Level 1 支持 session_id、task_id、repo_id、branch_id、tool_schema_id 等 hints；Level 2 是未来 SDK / MCP / LSP / Git adapter seam。

## 2. 模块目标

利用 workflow metadata 改善 prefix reuse、KV pinning、task-level scheduling、branch state reuse、tool schema reuse 和 cost per task，同时不破坏通用 serving。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Agent Metadata Runtime 不执行外部工具、不管理业务状态持久化、不要求 app 改协议。它只把可选 metadata 转成 AgentContext，提供给 scheduler、KV runtime、structured generation 和 observability。storage-backed Agent State 属于 Phase 2 seam 之后的其他系统，不进入 Phase 1。

## 6. 数据结构草案

AgentContext(session_id, task_id, repo_id, branch_id, tool_schema_id, workflow_step, priority)；StateLineage(parent_task, branch, reused_prefixes)；TaskCostTrace(task_id, model_calls, gpu_seconds, wall_ms, cost)。

## 7. 关键 API 草案

extract_agent_context(request_metadata)、attach_context(request_id, context)、lookup_task_state(context)、record_tool_boundary(event)、report_task_cost(task_id)。

## 8. 执行流程

普通请求无 metadata 时按 Level 0 处理；有 hints 时建立 AgentContext，scheduler 用它提升 prefix hit、pin 热 KV、避免跨任务抢占；structured generation 用 tool_schema_id 复用 mask/cache；observability 聚合 task-level 指标。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

Level 0/1 metadata parser、session/task/repo/branch/tool_schema hints、task-level metrics、prefix pin policy 和 Level 2 seam 文档。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
