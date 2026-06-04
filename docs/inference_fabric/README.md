# Inference Fabric 文档入口

本文档目录用于承载 Inference Fabric 的架构研究、Phase 1 范围、Codex handoff 和后续开发任务拆解。

## 当前项目定义

```text
Inference Fabric

面向未来的 Agent-aware、MoE-native、KV-memory-native、Distributed-native 通用 GPU 推理引擎。
```

本项目不是 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake 或 3FS 的 wrapper，也不把这些系统作为执行 backend。它们只作为竞品、benchmark 对照组和技术参考对象。

## Phase 1 固定范围

Phase 1 只做：

```text
Distributed memory-native GPU inference engine
```

Phase 1 包含：

- 通用 serving 能力：OpenAI-compatible API、streaming、structured output、tool calling、continuous batching、paged KV、prefix cache。
- Distributed Execution Graph：prefill、decode、attention、MoE expert、draft、verify 等执行单元可被 scheduler 管理。
- Distributed Memory Fabric：GPU HBM、peer GPU HBM、cross-node GPU HBM、CPU DRAM / pinned memory 的显式运行时内存层。
- In-memory KV/State Runtime：只管理运行时内存态 KV 和状态，不承诺持久化。
- MoE-native Runtime：expert 是系统级资源，而不是普通 kernel 内部细节。
- Agent Metadata Runtime：Agent 能力通过可选 metadata / SDK / adapter 增强，不强依赖外部 app 改协议。
- Speculative Draft/Verify Runtime：投机解码是 execution graph，而不是简单开关。
- GPU Kernel Runtime Strategy：通用 kernel 追平现有 SOTA，专用 kernel 服务于本系统独有路径。
- Benchmark / Profiling / Observability：必须 benchmark-first。

Phase 1 不包含：

- 持久化 KVStore 设计。
- 全闪 KV snapshot store。
- NVMe SSD KV restore。
- GDR KV storage I/O。
- 重启后的 KV 恢复。
- storage-backed Agent State。
- 3FS 替代品。

这些全部属于 Phase 2 seam。

## 硬件目标

Primary target:

```text
4 nodes × 8 × H20 GPUs × 96GB HBM
Total: 32 GPUs, about 3TB HBM
```

具体 NVLink / NVSwitch / PCIe / RDMA 拓扑必须通过 benchmark discovery 实测确认。

## 当前硬约束

```text
No CXL
No DPU / SmartNIC offload
No PIM
No FPGA
No 3FS dependency
No Phase-1 KVStore design
No storage hot decode path
```

## 推荐阅读顺序

1. `00_project_charter_cn.md`
2. `01_phase1_scope_distributed_memory_only_cn.md`
3. `02_first_principles_performance_model_cn.md`
4. `99_codex_master_prompt_v1_cn.md`
5. `docs/adr/*.md`

## 后续由 Codex 生成的完整文档树

Codex 后续应在本目录下生成或补全以下文档：

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

## Codex 接手原则

Codex 先补全文档树，不写实现代码。所有实现任务必须从文档、ADR、benchmark plan 和模块边界中派生。Codex 不得把 vLLM、SGLang、TensorRT-LLM、Dynamo、LMCache、Mooncake、3FS 设计成本项目 backend。