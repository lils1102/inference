# 14. GPU Kernel Runtime Strategy

## 1. 本文件结论

Phase 1 不承诺所有 kernel 全部自研并全面超过现有 SOTA。策略分两层：Baseline backend 使用 FlashAttention / FlashInfer / CUTLASS / Triton / vendor libraries 作为可替换底座；Custom breakthrough kernels 聚焦 Inference Fabric 独有路径。

## 2. 模块目标

用成熟 kernel 达到通用 serving 入场券，同时把研发火力集中到 shared-prefix attention、KV-owner-side attention、remote-prefix + local-suffix merge、MoE dispatch/combine fusion、speculative verify 和 KV movement。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Kernel Runtime 抽象 plan/build/run/profile 四个阶段。Baseline 覆盖 prefill/decode attention、paged KV attention、sampling、GEMM、norm、quantization。自研突破路径包括 shared-prefix attention、KV-owner-side attention、remote-prefix + local-suffix online softmax merge、MoE dispatch/combine fusion、expert-aware grouped GEMM、speculative verify kernel、structured-output mask + sampling fusion、KV append/copy/compact/migrate、H20-specific tuning。H20 具体调优不确定，需要 benchmark discovery 确认。

Baseline backend 分层：

- Attention / paged KV / sampling：FlashAttention、FlashInfer 或 vendor library 作为 baseline 或可替换后端。
- GEMM / grouped GEMM / epilogue：CUTLASS、Triton、vendor libraries 作为 baseline。
- Runtime integration：只封装低层 kernel/library，不把 vLLM、SGLang 或 TensorRT-LLM 作为 execution backend。

Custom breakthrough kernels 分层：

- KV locality：shared-prefix attention、KV-owner-side attention、remote-prefix + local-suffix online softmax merge。
- MoE locality：MoE dispatch/combine fusion、expert-aware grouped GEMM、hot expert batching。
- Speculative：verify kernel、ragged acceptance handling、KV transactional append/rollback。
- Structured output：mask + sampling fusion、tool-call constrained sampling。
- Memory fabric：KV append / copy / compact / migrate，按 HBM efficiency 和 KV bytes moved 计价。

## 6. 数据结构草案

KernelOp(name, backend, shape, dtype, layout, graph_safe)；KernelProfile(latency_us, bandwidth, occupancy, achieved_tflops, hbm_efficiency)；KernelFallback(preferred, fallback, correctness_tests)。

## 7. 关键 API 草案

select_kernel(op, shape, hardware_profile)、plan_attention(layout)、run_kernel(op_plan)、profile_kernel(op_plan)、register_custom_kernel(op, constraints)。

## 8. 执行流程

先用 baseline backend 跑通模型和 benchmark，profile 找到瓶颈，再对本项目独有路径做 custom kernel RFC、microbenchmark、correctness test 和 A/B test。未证明收益的 custom kernel 不进入默认路径。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

baseline backend adapter、kernel profiling harness、attention/sampling/KV movement microbench 和 2-3 个高价值 custom kernel RFC。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
