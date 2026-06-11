# 04. Hardware Topology：32×H20 Benchmark Discovery

## 1. 本文件结论

Primary hardware target 是 4 nodes × 8 × H20 GPUs × 96GB HBM，合计约 32 GPUs / 3TB HBM。NVLink、NVSwitch、PCIe、NUMA、RDMA、GPUDirect RDMA、P2P 和实际带宽/延迟全部不确定，需要 benchmark discovery 确认。

## 2. 模块目标

定义 H20 集群拓扑发现流程，避免 scheduler、parallelism、KV migration 和 MoE dispatch 依赖未实测假设。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

拓扑发现分为静态枚举和动态 microbenchmark。静态枚举包含 nvidia-smi topo -m、GPU/NUMA/NIC 绑定、驱动/CUDA/NCCL/OFED/UCX 版本；动态 benchmark 包含 HBM bandwidth、GPU memcpy、same-node P2P、PCIe、NCCL allreduce/alltoall、RDMA/GDR capability、多节点 latency。所有结果进入 TopologyProfile，scheduler 只消费 profile，不硬编码拓扑。

## 6. 数据结构草案

TopologyProfile(nodes, gpus, nics, numa, links, measured_bw, measured_latency, gdr_capability)；LinkMetric(src, dst, transport, bw_GBps, latency_us, p99_us, confidence)；GpuGroup(id, gpu_ids, local_rank, tp_candidate, ep_candidate)。

## 7. 关键 API 草案

discover_static_topology()、run_nccl_bench(op, group)、run_p2p_bench(pair)、run_rdma_probe(pair)、publish_topology_profile(profile)。

## 8. 执行流程

启动前跑 discovery，生成 profile；scheduler 读取 profile 生成 intra-node TP group、cross-node EP/DP/P-D candidates；上线后周期性抽样验证带宽和延迟漂移。任何硬件相关判断都写成“不确定，需要 benchmark discovery 确认”，直到 profile 有数据。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

一键生成 topology report、NCCL/P2P/RDMA/HBM benchmark 表和 scheduler 可读 JSON/YAML profile。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
