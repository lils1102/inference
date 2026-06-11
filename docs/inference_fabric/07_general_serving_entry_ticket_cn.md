# 07. General Serving Entry Ticket

## 1. 本文件结论

通用 serving 是入场券。没有 OpenAI-compatible API、streaming、chat/completion、structured output、tool calling、continuous batching、chunked prefill、paged KV、prefix cache、多节点部署和 observability，Agent-aware 与 MoE-native 差异化没有落地入口。

## 2. 模块目标

定义 Phase 1 最小通用 serving 能力和兼容层边界，确保 Inference Fabric 可替代主流推理栈的常规入口。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

API Compatibility Layer 接 OpenAI-style chat/completions/responses 形态，请求进入 Request Normalizer，输出 General Serving Context；metadata extractor 只提取可选 hints，不破坏普通客户端兼容。Serving 层不包含业务 Agent logic，只负责协议、streaming、auth seam、rate limit seam、request lifecycle、error model 和 observability。

## 6. 数据结构草案

ServingRequest(model, messages, prompt, tools, response_format, sampling, stream, metadata)；ServingContext(request_id, tokens, sampling_policy, output_contract, agent_context_ref)；StreamEvent(type, delta, usage, trace_id)。

## 7. 关键 API 草案

submit_chat_completion(request)、submit_completion(request)、stream_events(request_id)、cancel_request(request_id)、get_request_metrics(request_id)。

## 8. 执行流程

请求进入 API 层后做 schema validation、tokenization、chat template、metadata extraction、SLO classification，再构建 execution graph。输出阶段执行 detokenization、structured/tool validation、streaming 和 usage 统计。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

OpenAI-compatible chat/completion、streaming、basic tool call schema、JSON schema structured output、token usage、request cancellation 和 request-level metrics。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
