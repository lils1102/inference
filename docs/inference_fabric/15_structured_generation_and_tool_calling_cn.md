# 15. Structured Generation and Tool Calling

## 1. 本文件结论

Structured output 与 tool calling 是通用 serving 入场券，也是 Agent workload 的关键优化点。Phase 1 应支持 JSON schema / regex / grammar 类约束、tool schema caching、mask + sampling fusion 和 streaming-safe validation。

## 2. 模块目标

在保持 OpenAI-compatible 行为的同时降低 constrained decoding 的 TPOT 开销，提高 tool call 成功率，减少无效 token 和 Agent loop 重试。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Structured Runtime 分为 OutputContract、ConstraintCompiler、MaskCache、SamplingPlanner、ToolCallAssembler 和 Validator。tool_schema_id 可作为 metadata hint 触发 schema/mask cache reuse。约束系统必须与 speculative runtime 协调：draft token 也要受约束或在 verify 阶段安全拒绝。

## 6. 数据结构草案

OutputContract(type, schema, tool_defs, strictness)；CompiledConstraint(id, automaton, mask_layout, cache_key)；ToolCallState(name, args_buffer, validation_status)；SamplingPlan(logits_mask, topk, topp, temperature)。

## 7. 关键 API 草案

compile_constraint(contract)、lookup_mask_cache(cache_key)、apply_logits_mask(logits, state)、sample_token(plan)、validate_stream_delta(delta)、finalize_tool_call(state)。

## 8. 执行流程

请求解析 response_format/tools，编译或命中 constraint cache；每个 decode step 先计算 logits，再应用 mask，采样后更新 constraint state；streaming 输出需要增量 validation；tool call 完成后记录 task-level 成功/重试指标。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

JSON schema baseline、tool call assembler、mask cache、sampling metrics、speculative compatibility checklist 和 tool-call benchmark。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
