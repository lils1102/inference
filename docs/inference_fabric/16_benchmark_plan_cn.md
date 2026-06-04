# 16. Benchmark Plan

## 1. 本文件结论

Benchmark plan 是 Phase 1 的验收核心。必须覆盖竞品 baseline、H20 拓扑发现、通信/HBM/KV/MoE/speculative/Agent workflow，并用同等硬件、模型、精度、workload、SLO 比较。

## 2. 模块目标

产出可复现 benchmark suite，支撑 Target Pareto SOTA 判断和 scheduler/kernel 取舍。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Benchmark 分五层：硬件 discovery、microbenchmark、module benchmark、end-to-end serving benchmark、Agent task benchmark。Baseline 包含 vLLM、SGLang、Dynamo + vLLM/SGLang、LMCache、Mooncake、3FS competitor stack。3FS 只作为竞品组合栈组件，不进入 Inference Fabric 内部。

必须包含的 benchmark：

- Baseline：vLLM、SGLang、Dynamo + vLLM、Dynamo + SGLang、LMCache、Mooncake、3FS competitor baseline。
- H20 topology：`nvidia-smi topo -m`、GPU/NUMA/NIC 绑定、driver/CUDA/NCCL/UCX/OFED 版本。
- Communication：NCCL allreduce、NCCL alltoall、same-node P2P、cross-node RDMA / GPUDirect RDMA capability、multi-node latency、PCIe bandwidth。
- Memory：HBM bandwidth、GPU memory copy、KV append/copy/compact/migrate、HBM fragmentation。
- KV：Paged KV、exact prefix cache、high-prefix-reuse、KV-owner-side attention、remote-prefix + local-suffix attention、prefix recomputation 对照。
- MoE：expert dispatch latency、expert combine latency、expert imbalance、hot expert replication candidate、grouped GEMM efficiency。
- Speculative：draft cost、verify cost、accepted tokens per verify、rejection rate、wasted draft tokens、structured-output compatibility。
- Agent workflow：session/task/repo/branch/tool_schema metadata、prefix hit rate、agent task completion time、GPU seconds/task、cost per task。
- Serving：TTFT、TPOT、p99 latency、goodput under SLO、GPU utilization、HBM efficiency、KV bytes moved。

## 6. 数据结构草案

BenchmarkSuite(name, hardware, models, workloads, baselines, metrics)；RunConfig(seed, model, precision, parallelism, batch, slo)；MetricRecord(ttft, tpot, p99, goodput, gpu_util, hbm_eff, kv_bytes, prefix_hit, expert_latency, accepted_tokens, task_cost)。

## 7. 关键 API 草案

run_topology_discovery()、run_baseline(system, config)、run_module_bench(module, config)、compare_runs(a, b, metrics)、publish_report(run_set)。

## 8. 执行流程

先跑 nvidia-smi topo -m、NCCL allreduce/alltoall、P2P、HBM、PCIe、RDMA/GDR、multi-node latency；再跑 paged KV、prefix cache、MoE dispatch、speculative verify、structured output；最后跑 end-to-end ShareGPT/long-context/MoE/Agent workflow。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

一套可版本化配置、raw log 目录、指标表、baseline matrix 和不确定项清单。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
