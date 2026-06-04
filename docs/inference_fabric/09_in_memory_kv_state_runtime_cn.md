# 09. In-memory KV/State Runtime

## 1. 本文件结论

Phase 1 的 KV/State Runtime 只管理内存态 KV 与推理状态：GPU HBM、peer GPU HBM、cross-node GPU HBM materialization、CPU DRAM/pinned metadata。它不是持久化 KVStore，也不提供重启后恢复。

## 2. 模块目标

把 KV 从单 request cache 提升为可复用、可分支、可迁移、可 pin/evict、可被 scheduler 计价的 runtime state。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

KV runtime 维护 KVBlockTable、PrefixIndex、SessionState、BranchState 和 StateLineage。Prefix cache 先做 exact prefix baseline，再扩展 shared-prefix 和 branch-aware reuse。Copy-on-write 用于 speculative、parallel sampling 和 Agent branch。Hot decode KV 必须在 HBM。

## 6. 数据结构草案

KVBlock(block_id, layer, head_range, token_range, dtype, owner, bytes)；PrefixEntry(hash, token_span, block_ids, hit_count, owner)；SessionState(session_id, active_prefixes, pinned_blocks, ttl)；BranchState(parent, delta_blocks, cow_refs)。

## 7. 关键 API 草案

lookup_prefix(tokens, metadata)、append_kv(request_id, blocks)、fork_state(parent_state)、materialize_state(state_ref, gpu_group)、release_state(state_ref)、report_prefix_hit(entry)。

## 8. 执行流程

请求 tokenization 后查 PrefixIndex；命中则返回 state_ref 和 owner；scheduler 选择复用/迁移/owner-side attention/重算；decode append 新 KV blocks；结束后按 metadata、TTL、HBM pressure 和 hit rate 决定 pin 或 evict。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

paged KV block table、exact prefix cache、session_id/task_id hints、copy-on-write branch、pin/unpin/evict 和 prefix hit metrics。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
