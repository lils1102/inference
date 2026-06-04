# 99. Codex Master Prompt v1

把下面整段提示词交给 Codex。Codex 的任务是补全架构文档树，不要写实现代码。

---

你是一个大模型推理系统、分布式系统、GPU 高性能计算、NCCL/RDMA、MoE 推理、KVCache 系统、Agent 基础设施、CUDA kernel、benchmark/profiling 方向的技术研究员和系统架构师。

你正在 `lils1102/inference` 仓库中工作。请基于仓库已有 README、`docs/inference_fabric/` 和 `docs/adr/` 中的约束，补全 Inference Fabric 的 Phase 1 架构文档。

## 1. 项目定义

项目名称：

```text
Inference Fabric
```

项目定位：

```text
面向未来的 Agent-aware、MoE-native、KV-memory-native、Distributed-native 通用 GPU 推理引擎。
```

本项目不是 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake 或 3FS 的 wrapper，也不把这些系统作为执行 backend。它们只能作为竞品、benchmark 对照组和技术参考对象。

Phase 1 目标是：

```text
Distributed memory-native GPU inference engine
```

也就是先完成：通用 serving、distributed execution graph、distributed memory fabric、in-memory KV/State Runtime、MoE-native Runtime、Agent metadata runtime、draft/verify runtime、GPU kernel strategy、benchmark/profiling 基础设施。

## 2. 硬件目标

Primary target:

```text
4 nodes × 8 × H20 GPUs × 96GB HBM
Total: 32 GPUs, about 3TB HBM
```

不要静态假设 NVLink/NVSwitch/PCIe/RDMA 拓扑。文档必须要求通过 benchmark discovery 实测：`nvidia-smi topo -m`、NCCL tests、P2P bandwidth、RDMA/GDR capability、HBM bandwidth 等。

## 3. Phase 1 硬约束

必须严格遵守：

```text
No CXL
No DPU / SmartNIC offload
No PIM
No FPGA
No 3FS dependency
No Phase-1 KVStore design
No NVMe SSD KV restore in Phase 1
No GDR KV storage I/O in Phase 1
No storage-backed Agent State in Phase 1
No storage hot decode path
```

3FS 只能作为竞品组合栈的一部分出现在 competitor analysis 中，例如：

```text
Dynamo + vLLM/SGLang + LMCache + Mooncake + 3FS
```

不得把 3FS 设计为本项目内部组件。

自研分布式全闪 KVCache Store 也不要在 Phase 1 中设计。它是 Phase 2 seam，属于未来基于 NVMe SSD 的持久化存储层，不是内存层。

## 4. 分布式内存层定义

Phase 1 的 Distributed Memory Fabric 是显式运行时内存层，不是 CXL 式透明远端内存，也不是 NVMe SSD 存储层。

它覆盖：

```text
L0: Local GPU HBM
L1: Same-node peer GPU HBM
L2: Cross-node GPU HBM through RDMA / GPUDirect RDMA / NCCL / P2P
L3: CPU DRAM / pinned memory / metadata / staging buffers
```

它负责：

- GPU HBM hot KV 管理。
- Peer GPU KV migration。
- Cross-node KV materialization。
- Activation dispatch。
- Expert output return。
- KV placement / migration / compaction。
- In-memory KV pin / unpin / eviction。
- KV-owner-side attention execution placement。

它不负责：

- 持久化 checkpoint。
- NVMe KV restore。
- storage-backed failure recovery。
- 每 token 从 SSD/KVStore 读取 decode hot KV。

Hot decode KV 必须在 GPU HBM。

## 5. 系统中心

系统中心不是 Agent Runtime，也不是单机 engine，而是：

```text
Distributed Execution Graph
```

建议架构：

```text
API Compatibility Layer
  ↓
Request Normalizer / Metadata Extractor
  ├── General Serving Context
  └── Optional Agent Context
        ↓
Global Distributed Scheduler
        ↓
Distributed Execution Graph
        ↓
Model Runtime / KV Memory Runtime / MoE Runtime / Speculative Runtime / Communication Fabric / GPU Kernel Runtime
```

Agent Runtime 应被写成 Agent Metadata Runtime 或 Agent-aware Context Runtime。它不能强制外部 Agent app 改协议。

Agent 接入分三档：

```text
Level 0: 普通 OpenAI API，无侵入。
Level 1: metadata hints，例如 session_id、task_id、repo_id、branch_id、tool_schema_id。
Level 2: 未来 SDK / MCP / LSP / Git adapter seam。
```

## 6. 必须补全的文档

请在 `docs/inference_fabric/` 下补全以下文档。如果文件已存在，请在不破坏现有约束的前提下扩展；如果不存在，请创建。

```text
03_sota_baseline_truth_layer_cn.md
04_hardware_topology_32xh20_cn.md
05_competitor_stack_analysis_cn.md
06_distributed_execution_graph_cn.md
07_general_serving_entry_ticket_cn.md
08_distributed_memory_fabric_cn.md
09_in_memory_kv_state_runtime_cn.md
10_moe_native_runtime_cn.md
11_distributed_scheduler_cn.md
12_speculative_runtime_cn.md
13_agent_metadata_runtime_cn.md
14_gpu_kernel_strategy_cn.md
15_structured_generation_and_tool_calling_cn.md
16_benchmark_plan_cn.md
17_phase2_persistent_kvstore_seam_cn.md
18_risks_and_counterarguments_cn.md
19_open_source_and_ecosystem_strategy_cn.md
20_module_task_breakdown_for_codex_cn.md
```

每份文档必须包含：

1. 本文件结论。
2. 模块目标。
3. 非目标。
4. 第一性原理瓶颈。
5. 核心设计。
6. 数据结构草案。
7. 关键 API 草案。
8. 执行流程。
9. 性能瓶颈。
10. Benchmark / profiling 指标。
11. MVP 范围。
12. 风险。
13. Implementation Notes。

## 7. 必须补全的 ADR

请在 `docs/adr/` 下补全或创建：

```text
0001-next_generation_distributed_inference_engine.md
0002-general_serving_is_entry_ticket.md
0003-agent_native_is_optional_metadata_layer.md
0004-phase1_distributed_memory_only_no_kvstore.md
0005-no_cxl_no_dpu_no_3fs_dependency.md
0006-kvstore_is_future_persistent_storage_not_memory.md
0007-moe_expert_as_system_resource.md
0008-active_kv_as_execution_mode.md
0009-intra_node_tp_cross_node_ep_default.md
0010-pareto_sota_not_absolute_sota.md
```

每个 ADR 使用：

```text
# ADR-NNNN: Title

Status: Accepted / Proposed

## Context
## Decision
## Consequences
## Alternatives Considered
## Implementation Notes
```

## 8. 竞品与参考系统

文档必须系统比较：

```text
vLLM
SGLang
TensorRT-LLM
NVIDIA Dynamo
LMCache
Mooncake
3FS
FlashAttention
FlashInfer
DeepEP / NIXL / NCCL related stack
```

比较重点：

- 通用 serving 能力。
- Paged KV / prefix cache / structured output。
- Speculative decoding。
- P/D disaggregation。
- MoE expert parallel。
- KV offload / transfer / storage。
- Multi-node deployment。
- Agent/workflow state awareness。
- 我们的可超越点。
- 我们不应正面硬刚的点。

## 9. GPU kernel 策略

不要承诺 Phase 1 所有 kernel 全部自研且全面超过现有 SOTA。

必须写成两层：

```text
Baseline backend:
  FlashAttention / FlashInfer / CUTLASS / Triton / vendor libraries as baseline or replaceable backend.

Custom breakthrough kernels:
  shared-prefix attention
  KV-owner-side attention
  remote-prefix + local-suffix online softmax merge
  MoE dispatch/combine fusion
  expert-aware grouped GEMM
  speculative verify kernel
  structured-output mask + sampling fusion
  KV append / copy / compact / migrate
  H20-specific tuning
```

本项目不使用 vLLM/SGLang/TensorRT-LLM 作为 backend，但可以参考或临时 benchmark 低层 kernel/library。

## 10. Scheduler 分阶段

不要把 scheduler 写成一开始就全局最优。

必须分阶段：

```text
Scheduler V0: continuous batching + chunked prefill + basic TP/EP/DP placement
Scheduler V1: KV locality + prefix cache + HBM pressure
Scheduler V2: MoE expert locality + expert queue + hot expert replication
Scheduler V3: draft/verify scheduling + speculative cost model
Scheduler V4: Agent metadata / workflow-aware scheduling
Scheduler V5: learned cost model / auto-tuning
```

## 11. Benchmark-first

必须生成 benchmark plan。至少包括：

- vLLM baseline。
- SGLang baseline。
- Dynamo + vLLM / SGLang baseline。
- LMCache baseline。
- Mooncake baseline。
- 3FS competitor baseline。
- H20 topology benchmark。
- NCCL allreduce / alltoall。
- RDMA / GDR capability。
- HBM bandwidth。
- Paged KV benchmark。
- Prefix cache benchmark。
- MoE expert dispatch benchmark。
- Speculative verify benchmark。
- Agent metadata workflow benchmark。
- TTFT / TPOT / p99 / goodput under SLO。
- GPU seconds / task。

## 12. 输出要求

- 中文 Markdown。
- 不写实现代码。
- 不发明未确认事实。
- 对不确定信息标注“不确定，需要 benchmark 或调研确认”。
- 不要把 Phase 2 的 KVStore 写成 Phase 1 实现。
- 不要把 NVMe / KVStore / 3FS 放进 decode hot path。
- 每份文档末尾给出 Implementation Notes。
- 所有设计必须能映射到 benchmark 指标。

## 13. 完成标准

Codex 完成后，仓库应具备：

1. 完整 `docs/inference_fabric/` 文档树。
2. 完整 `docs/adr/` 架构决策记录。
3. 清晰的 Phase 1 / Phase 2 边界。
4. 可执行的 benchmark plan。
5. 可派生后续开发任务的 module breakdown。
6. 不包含任何 Phase 1 禁止事项。