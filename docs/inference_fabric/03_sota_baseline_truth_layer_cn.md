# 03. SOTA Baseline Truth Layer

## 1. 本文件结论

SOTA baseline truth layer 负责把竞品能力、版本、配置、硬件、模型、workload 和指标固定成可复现实验，不用二手印象替代 benchmark。任何“超过 SOTA”的表述只允许绑定到明确 benchmark suite。

## 2. 模块目标

建立 vLLM、SGLang、TensorRT-LLM、Dynamo、LMCache、Mooncake、3FS、FlashAttention、FlashInfer、DeepEP / NIXL / NCCL 的事实记录和对照配置。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

每个 baseline 记录官方文档链接、版本、commit、镜像、模型、精度、parallelism、KV 配置、prefix/speculative/structured output 开关、部署拓扑和 SLO。Dynamo、LMCache、Mooncake、3FS 只能作为竞品组合栈组件，不作为本项目 backend。

竞品事实源应优先使用官方文档、官方仓库或论文：

- vLLM：PagedAttention、automatic prefix caching、structured outputs、speculative decoding 和 OpenAI-compatible serving，以 <https://docs.vllm.ai/> 与 vLLM 论文/仓库为准。
- SGLang：RadixAttention、constrained/structured decoding、speculative decoding 和 serving runtime，以 <https://docs.sglang.ai/> 为准。
- TensorRT-LLM：in-flight batching、paged KV cache、quantization、speculative decoding、Triton backend 和 vendor optimized kernels，以 <https://nvidia.github.io/TensorRT-LLM/> 为准。
- NVIDIA Dynamo：disaggregated serving、KV-aware routing、NIXL transfer、RDMA dependency 和 distributed serving，以 <https://docs.nvidia.com/dynamo/> 为准。
- LMCache：KV cache offload / transfer / connector 能力，以 <https://docs.lmcache.ai/> 和官方技术报告为准。
- Mooncake：KVCache-centric disaggregated architecture、Mooncake Store 和 transfer/storage 设计，以 <https://kvcache-ai.github.io/Mooncake/> 与 <https://github.com/kvcache-ai/Mooncake> 为准。
- 3FS：只作为 competitor storage layer 进入组合式 baseline，以官方仓库/论文为准；不得作为 Inference Fabric dependency。
- FlashAttention、FlashInfer、CUTLASS、Triton、vendor libraries：只作为低层 kernel baseline 或可替换 backend 参考，不作为 serving engine backend。
- DeepEP / NIXL / NCCL：作为 expert parallel、KV transfer 和 collectives 的通信参考，能力和收益不确定，需要在目标 H20 集群实测确认。

能力矩阵至少包含：通用 serving、Paged KV / prefix cache / structured output、speculative decoding、P/D disaggregation、MoE expert parallel、KV offload / transfer / storage、multi-node deployment、Agent/workflow state awareness、我们可超越点、我们不应正面硬刚的点。

## 6. 数据结构草案

BaselineSystem(name, role, version, source_url, config_uri)；BenchmarkRun(baseline, hardware, model, workload, metrics, raw_logs)；CapabilityMatrix(serving, paged_kv, prefix_cache, structured_output, spec_decode, pd_disagg, moe_ep, kv_transfer, multi_node, agent_awareness)。

## 7. 关键 API 草案

register_baseline(system)、freeze_config(run_id)、compare_against_fabric(run_id, metric_set)、mark_uncertain(fact, reason)。

## 8. 执行流程

先从官方文档/仓库记录能力，再在同等硬件上跑基线，最后只把实测结果写进 SOTA claim。若版本、配置或硬件不同，结论必须标注不确定，需要 benchmark 或调研确认。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

一张能力矩阵、一套 baseline 运行配置、一套 raw metrics 目录约定和 SOTA claim 模板。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
