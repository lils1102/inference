# 05. Competitor Stack Analysis

## 1. 本文件结论

当前业界组合式推理栈可以抽象为 Dynamo + vLLM/SGLang/TensorRT-LLM + LMCache + Mooncake + 3FS / storage layer。Inference Fabric 不拼装这些系统，而是在通用 serving 基线之上原生实现分布式执行、内存态 KV、MoE、speculative 和 Agent metadata。

## 2. 模块目标

系统比较竞品能力、可借鉴点、应避免正面硬刚的点和 Inference Fabric 的可超越方向。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

vLLM 代表 PagedAttention、continuous batching、prefix cache 和 OpenAI-compatible serving 基线；SGLang 代表 RadixAttention、structured/constrained generation 和低延迟 serving；TensorRT-LLM 代表 vendor optimized engine、in-flight batching、paged KV、quantization 和高性能 kernel；Dynamo 代表 distributed serving、P/D disaggregation、KV-aware routing 和 NIXL transfer；LMCache/Mooncake/3FS 代表 KV offload/transfer/storage 方向的竞品组件；FlashAttention/FlashInfer/CUTLASS/Triton/vendor libraries 作为低层 kernel baseline；DeepEP/NIXL/NCCL 作为 communication reference。

| 系统 | 竞品层次 | 应比较能力 | 我们的可超越点 | 不应正面硬刚的点 |
|---|---|---|---|---|
| vLLM | 通用 serving engine | OpenAI API、continuous batching、PagedAttention、prefix cache、speculative、structured output、多 GPU serving | 分布式执行图、KV-owner-side attention、Agent task cost、MoE/system-level scheduling | 已成熟的通用模型兼容面和社区生态 |
| SGLang | serving/runtime/programming stack | RadixAttention、structured/constrained generation、speculative、低延迟 serving | Agent metadata 与 KV/state lineage、distributed memory-native runtime | 其成熟 structured generation ergonomics |
| TensorRT-LLM | vendor optimized engine | in-flight batching、paged KV、quantization、CUDA Graph、Triton backend、high-performance kernels | 开放 execution graph、Agent/MoE/KV scheduler 联合优化 | vendor 单 kernel 峰值和闭环硬件优化 |
| NVIDIA Dynamo | distributed serving layer | P/D disaggregation、KV-aware routing、NIXL KV transfer、RDMA deployment | 原生一体化 graph + memory + MoE + Agent metadata | 大规模生产编排生态和 NVIDIA 集成 |
| LMCache | KV cache layer | KV offload、connector、pin/lookup/move/compression、CPU/storage/network tier | Phase 1 只做 in-memory hot path，避免 storage hot path latency | 持久化/离线 KV reuse 的成熟经验 |
| Mooncake | KVCache-centric disaggregated stack | P/D 分离、KV transfer、Mooncake Store、KV-centric scheduling | GPU HBM hot KV 和 owner-side execution 的 runtime-first 路径 | storage/transfer system 的完整性 |
| 3FS | competitor storage component | 作为组合式 baseline 的 storage layer | 不在 Phase 1 内部依赖，避免范围膨胀 | 分布式文件/存储系统能力 |
| FlashAttention / FlashInfer | kernel baseline | attention、paged KV、sampling、append、layout 支持 | shared-prefix、remote-prefix + local-suffix merge、H20-specific workload tuning | 通用 attention kernel 峰值 |
| DeepEP / NIXL / NCCL | communication reference | EP dispatch/combine、KV transfer、collectives、all-to-all/allreduce | scheduler 与 workload-aware placement 联动 | 底层通信库本身的通用优化 |

上述能力不是静态胜负结论。所有“强/弱/可超越”都需要同等硬件、模型、精度、workload 和 SLO 下的 benchmark 确认。

## 6. 数据结构草案

CompetitorProfile(name, layer, strengths, gaps, baseline_role, do_not_compete_directly_on)；StackConfig(frontend, engine, kv_layer, storage_layer, communication, metrics)；DifferentiationHypothesis(workload, mechanism, expected_metric_gain, required_benchmark)。

## 7. 关键 API 草案

build_competitor_matrix()、define_baseline_stack(stack)、record_gap_analysis(profile)、link_benchmark_to_hypothesis(hypothesis, run_id)。

## 8. 执行流程

先对每个竞品记录能力，再组合成 baseline stack，在同等硬件和 workload 下跑 benchmark。对我们不该硬刚的点，例如成熟模型兼容覆盖、vendor 单 kernel 峰值、生态集成，采用 baseline backend 或分阶段追赶。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

竞品矩阵、组合式 baseline plan、可超越点列表和不硬刚清单。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
