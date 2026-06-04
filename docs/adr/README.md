# Architecture Decision Records

本目录记录 Inference Fabric 的核心架构决策。ADR 是 Codex 后续生成文档和开发任务时必须遵守的硬约束。

## ADR 列表

| ADR | 状态 | 决策 |
|---|---|---|
| `0001-next_generation_distributed_inference_engine.md` | Accepted | 项目定位为下一代原生分布式通用 GPU 推理引擎 |
| `0002-general_serving_is_entry_ticket.md` | Accepted | 通用 serving 是入场券，Agent-aware 是增强层 |
| `0003-agent_native_is_optional_metadata_layer.md` | Accepted | Agent-aware 能力通过可选 metadata / SDK / adapter 增强，不强迫外部 app 改协议 |
| `0004-phase1_distributed_memory_only_no_kvstore.md` | Accepted | Phase 1 只做分布式内存层，不设计持久化 KVStore |
| `0005-no_cxl_no_dpu_no_3fs_dependency.md` | Accepted | 不依赖 CXL、DPU、PIM、FPGA、3FS |
| `0006-kvstore_is_future_persistent_storage_not_memory.md` | Accepted | 未来 KVStore 是持久化存储，不是内存层 |
| `0007-moe_expert_as_system_resource.md` | Accepted | MoE expert 是系统级资源 |
| `0008-active_kv_as_execution_mode.md` | Accepted | Active KV 定义为 KV-owner-side attention execution mode |
| `0009-intra_node_tp_cross_node_ep_default.md` | Proposed | 默认假设 intra-node TP、cross-node EP/DP/P-D/KV-state sharding，需 benchmark 验证 |
| `0010-pareto_sota_not_absolute_sota.md` | Accepted | 目标是 Target Pareto SOTA，不是所有场景绝对第一 |

## Codex 使用规则

Codex 后续生成设计文档时：

1. 必须遵守所有 Accepted ADR。
2. `0009` 是 Proposed，必须写成 benchmark-driven default，而不是绝对规则。
3. 不得重新把 CXL、DPU、3FS、Phase-1 KVStore 写入 Phase 1 主架构。
4. 不得把全闪 KVStore 描述为内存层或 decode hot path。
5. 不得把 vLLM、SGLang、TensorRT-LLM、Dynamo、LMCache、Mooncake、3FS 设计为本项目 backend。

## 更新规则

如需修改核心边界，必须新增或更新 ADR，而不是只修改普通文档。