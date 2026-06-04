# 98. Pre-Codex Handoff Status

## 结论

Inference Fabric 当前已达到可以交给 Codex 继续补全 Phase 1 架构文档树的状态。

Codex 下一步应当只做文档工作，不写实现代码。

## 已完成

### 1. 根 README 已更新

根 README 已经固定以下内容：

- 项目定位：Agent-aware、MoE-native、KV-memory-native、Distributed-native 的通用 GPU 推理引擎。
- 目标硬件：4 nodes × 8 × H20 GPUs × 96GB HBM。
- Phase 1 范围：distributed memory-native GPU inference engine。
- Phase 1 不包含：持久化 KVStore、全闪 KV snapshot store、NVMe SSD KV restore、GDR KV storage I/O、重启后 KV 恢复、storage-backed Agent State、3FS 替代品。
- No CXL / No DPU / No PIM / No FPGA / No 3FS dependency / No Phase-1 KVStore design。
- Distributed Memory Fabric 是显式运行时内存层，不是 CXL、不是 NVMe、不是持久化存储。
- Phase 2 seam 才可能接入自研分布式全闪 KVCache Store。

### 2. Phase 1 文档入口已创建

已创建：

```text
docs/inference_fabric/README.md
```

该文件说明了：

- Phase 1 范围。
- 硬件目标。
- 当前硬约束。
- 推荐阅读顺序。
- Codex 后续应补全的文档树。
- Codex 接手原则。

### 3. Project Charter 已创建

已创建：

```text
docs/inference_fabric/00_project_charter_cn.md
```

该文件固定了：

- 项目定位。
- 非目标。
- 当前目标硬件。
- 项目核心假设。
- Phase 1 交付物。
- Phase 1 不交付项。
- 第一性原理判断。

### 4. Phase 1 Scope 已创建

已创建：

```text
docs/inference_fabric/01_phase1_scope_distributed_memory_only_cn.md
```

该文件明确：

- Phase 1 只做 distributed memory-native GPU inference engine。
- Phase 1 包含通用 serving、Distributed Execution Graph、Distributed Memory Fabric、In-memory KV/State Runtime、MoE Runtime、Agent Metadata Runtime、Speculative Runtime。
- Phase 1 不做 persistent KVStore 或 storage-backed state。

### 5. First-principles Performance Model 已创建

已创建：

```text
docs/inference_fabric/02_first_principles_performance_model_cn.md
```

该文件固定了：

- Total Cost 模型。
- Prefill / decode 差异。
- TTFT 模型。
- TPOT 模型。
- Speculative Runtime 模型。
- MoE 性能模型。
- Distributed Memory Fabric cost model。
- Agent Task Cost 模型。
- Benchmark-first 原则。

### 6. Codex Master Prompt 已创建

已创建：

```text
docs/inference_fabric/99_codex_master_prompt_v1_cn.md
```

这是交给 Codex 的主提示词。Codex 应严格按该文件执行。

### 7. ADR 已创建

已创建：

```text
docs/adr/0001-next_generation_distributed_inference_engine.md
docs/adr/0002-general_serving_is_entry_ticket.md
docs/adr/0003-agent_native_is_optional_metadata_layer.md
docs/adr/0004-phase1_distributed_memory_only_no_kvstore.md
docs/adr/0005-no_cxl_no_dpu_no_3fs_dependency.md
docs/adr/0006-kvstore_is_future_persistent_storage_not_memory.md
docs/adr/0007-moe_expert_as_system_resource.md
docs/adr/0008-active_kv_as_execution_mode.md
docs/adr/0009-intra_node_tp_cross_node_ep_default.md
docs/adr/0010-pareto_sota_not_absolute_sota.md
```

这些 ADR 固定了 Phase 1 的关键架构决策。

## Codex 下一步应该做什么

Codex 下一步应根据：

```text
docs/inference_fabric/99_codex_master_prompt_v1_cn.md
```

补全以下文档：

```text
docs/inference_fabric/03_sota_baseline_truth_layer_cn.md
docs/inference_fabric/04_hardware_topology_32xh20_cn.md
docs/inference_fabric/05_competitor_stack_analysis_cn.md
docs/inference_fabric/06_distributed_execution_graph_cn.md
docs/inference_fabric/07_general_serving_entry_ticket_cn.md
docs/inference_fabric/08_distributed_memory_fabric_cn.md
docs/inference_fabric/09_in_memory_kv_state_runtime_cn.md
docs/inference_fabric/10_moe_native_runtime_cn.md
docs/inference_fabric/11_distributed_scheduler_cn.md
docs/inference_fabric/12_speculative_runtime_cn.md
docs/inference_fabric/13_agent_metadata_runtime_cn.md
docs/inference_fabric/14_gpu_kernel_strategy_cn.md
docs/inference_fabric/15_structured_generation_and_tool_calling_cn.md
docs/inference_fabric/16_benchmark_plan_cn.md
docs/inference_fabric/17_phase2_persistent_kvstore_seam_cn.md
docs/inference_fabric/18_risks_and_counterarguments_cn.md
docs/inference_fabric/19_open_source_and_ecosystem_strategy_cn.md
docs/inference_fabric/20_module_task_breakdown_for_codex_cn.md
```

## Codex 不应该做什么

Codex 不应：

- 写实现代码。
- 设计 Phase 1 KVStore。
- 把 NVMe SSD / all-flash KVStore 当作内存层。
- 把 3FS 设计成本项目内部组件。
- 引入 CXL / DPU / SmartNIC offload / PIM / FPGA 依赖。
- 把 vLLM、SGLang、TensorRT-LLM、Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- 把 Agent Runtime 写成强制外部 app 改协议。
- 承诺所有指标绝对 SOTA。

## Codex 完成标准

Codex 完成后，应具备：

1. 完整 Phase 1 架构文档树。
2. 每份文档包含目标、非目标、第一性原理、核心设计、数据结构、API 草案、执行流程、性能瓶颈、benchmark、MVP、风险、Implementation Notes。
3. Benchmark plan 能直接用于后续实验。
4. Module task breakdown 能直接派生后续开发任务。
5. 所有文档与 Phase 1 边界一致。

## 当前状态

```text
Status: Ready for Codex documentation expansion.
Scope: Phase 1 docs only.
Code generation: Not yet.
```
