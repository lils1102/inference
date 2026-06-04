# 02. First-principles Performance Model

## 结论

Inference Fabric 的所有架构设计都必须服从第一性原理性能模型。系统目标不是堆功能，而是最小化端到端推理成本：

```text
Total Cost =
  queueing_delay
+ prefill_cost
+ runtime_KV_state_cost
+ decode_steps × per_decode_step_cost
+ communication_cost
+ scheduler_overhead
+ tool_or_agent_wait
```

Phase 1 不包含持久化 KVStore，因此性能模型只覆盖 runtime memory state，不覆盖 NVMe / 全闪 KV restore 热路径。

## 1. Prefill 与 Decode 的本质差异

### Prefill

Prefill 处理整个 prompt，通常具有更高并行度，主要消耗：

- GPU compute。
- HBM bandwidth。
- Attention compute / KV write。
- Large batch GEMM。

Prefill 优化重点：

- Chunked prefill。
- Prefill / decode interference control。
- Prefix cache hit。
- Shared-prefix prefill / attention。
- P/D disaggregation。

### Decode

Decode 每步通常只新增一个或少量 token，容易变成 memory-bound 或 communication-bound。

Decode 每步成本近似为：

```text
per_decode_step_cost =
max(
  active_weight_read / HBM_bandwidth,
  attention_KV_read / HBM_bandwidth,
  expert_dispatch / interconnect_bandwidth,
  GEMM_compute / tensor_core_throughput,
  all_reduce_or_all_to_all_sync,
  kernel_launch_and_sync,
  scheduler_overhead
)
```

对于 MoE 模型，decode 还包括：

- router / top-k。
- expert dispatch。
- grouped GEMM。
- expert combine。
- expert imbalance。
- cross-node all-to-all。

## 2. TTFT 模型

TTFT 主要受以下因素影响：

```text
TTFT =
  queueing_delay
+ prompt_processing_time
+ prefix_cache_lookup
+ KV_materialization_or_recompute
+ prefill_execution
+ first_decode_step
```

Phase 1 可优化：

- prefix cache。
- in-memory KV reuse。
- hot/warm KV pin。
- shared-prefix attention。
- chunked prefill。
- scheduler locality。

Phase 1 不优化或不承诺：

- persistent KV restore from NVMe / all-flash KVStore。
- restart-time KV recovery。
- long-term cold state resume。

## 3. TPOT 模型

TPOT 主要受 decode step 控制：

```text
TPOT ≈ per_decode_step_cost
```

可优化方向：

- continuous batching。
- decode batching。
- CUDA Graph / persistent kernels。
- MoE expert batching。
- expert locality。
- KV locality。
- speculative accepted tokens per verify。
- structured-output optimized sampling。

不应假设：

- 从 SSD / KVStore 每 token 读取 KV 能降低 TPOT。
- 远端内存透明访问可以替代 HBM。

## 4. Speculative Runtime 模型

Speculative decoding 的有效可见 TPOT：

```text
visible_TPOT ≈ target_forward_TPOT / accepted_tokens_per_target_forward
```

或：

```text
visible_tokens_per_second ≈ target_forwards_per_second × accepted_tokens_per_forward
```

系统应追踪：

- draft cost。
- verify cost。
- accepted tokens per verify。
- rejection rate。
- wasted draft tokens。
- tree verification overhead。
- target verifier queueing。

## 5. MoE 性能模型

MoE 每层成本：

```text
MoE_layer_cost =
  router_cost
+ expert_dispatch_cost
+ expert_queue_delay
+ expert_compute_cost
+ expert_combine_cost
+ synchronization_cost
```

MoE-native Runtime 的目标是降低：

- expert dispatch latency。
- expert imbalance。
- small expert GEMM inefficiency。
- cross-node all-to-all overhead。
- hot expert p99 queueing。

## 6. Distributed Memory Fabric 模型

Phase 1 的内存层是显式运行时，不是透明远端内存。

关键操作：

```text
place(KVBlock, GPU)
move(KVBlock, src, dst)
pin(KVBlock)
evict(KVBlock)
compact(GPU)
remote_attention(Q, KVOwner)
```

每个操作都必须有 cost model：

```text
Cost_move_KV = KV_bytes / effective_transfer_bandwidth + sync_cost
Cost_remote_attention = Q_bytes / bw + remote_attention_compute + O_bytes / bw + merge_cost + sync_cost
Cost_recompute = prefill_or_attention_compute_cost
```

Scheduler 必须在以下策略中选择：

- local attention。
- KV migration then local attention。
- KV-owner-side attention。
- prefix recomputation。
- keep current placement。

## 7. Agent Task Cost 模型

Agent / workflow 不是单个 request。Agent task 成本：

```text
AgentTaskCost =
  model_call_cost
+ repeated_context_cost
+ tool_latency
+ branch_exploration_cost
+ verifier_cost
+ queueing_delay
+ state_retention_cost
```

目标指标：

- task completion time。
- GPU seconds / task。
- cost per task。
- prefix hit rate。
- useful accepted patch / verifier call。
- tool-call loop latency。

## 8. Benchmark-first 原则

任何设计必须能够映射到 benchmark：

- TTFT。
- TPOT。
- goodput under SLO。
- p99 latency。
- HBM utilization。
- KV bytes moved。
- expert dispatch latency。
- expert imbalance。
- accepted tokens per verify。
- Agent task completion time。

## Implementation Notes

- Codex 后续生成模块设计时，必须把每个模块映射到本性能模型。
- 不允许用“理论更优”替代 benchmark 指标。
- Phase 1 不得把持久化 KVStore、NVMe restore、3FS storage path 写入热路径性能模型。