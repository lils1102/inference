# 01. Phase 1 Scope：Distributed Memory Only

## 结论

Phase 1 的目标是构建：

```text
Distributed memory-native GPU inference engine
```

也就是先完成一个通用、原生分布式、GPU-first 的推理引擎；它具备分布式内存层、内存态 KV/State Runtime、MoE-native Runtime、Agent-aware metadata runtime、draft/verify runtime 和 benchmark/profiling 基础设施。

Phase 1 不设计持久化 KVStore，不设计 NVMe SSD KV restore，不设计 3FS 替代品。

## Phase 1 包含

### 1. 通用 Serving 能力

必须支持：

- OpenAI-compatible API。
- Streaming generation。
- Chat completion / completion。
- Structured output / tool calling。
- HF safetensors loader。
- tokenizer.json / chat template。
- Continuous batching。
- Chunked prefill。
- Paged KV baseline。
- Prefix cache baseline。
- Speculative decoding baseline。
- Multi-node deployment baseline。
- Observability / profiling / metrics。

### 2. Distributed Execution Graph

系统中心不是 Agent Runtime，也不是单机 engine，而是 Distributed Execution Graph。

Execution graph 应能表达：

- Prefill。
- Decode。
- Attention。
- MoE expert dispatch / compute / combine。
- Draft。
- Verify。
- Structured output mask / sampling。
- KV migration / compaction。
- Agent metadata-driven branch / workflow hint。

### 3. Distributed Memory Fabric

Phase 1 的运行时内存层覆盖：

```text
L0: Local GPU HBM
L1: Same-node peer GPU HBM
L2: Cross-node GPU HBM through RDMA / GPUDirect RDMA / NCCL / P2P
L3: CPU DRAM / pinned memory / metadata / staging buffers
```

它是显式调度、显式迁移、拓扑感知的 runtime memory fabric，不是 CXL 式透明远端内存。

### 4. In-memory KV/State Runtime

Phase 1 只支持内存态状态：

- GPU HBM hot KV。
- Peer GPU KV。
- Cross-node KV materialization。
- CPU DRAM / pinned warm metadata。
- Prefix index。
- KV block table。
- Copy-on-write KV。
- Session KV。
- Branch KV。
- Tool-schema KV。
- KV pin / unpin / eviction。
- KV migration / compaction。

不支持持久化 KV snapshot。

### 5. MoE-native Runtime

MoE expert 是系统级资源。Phase 1 需要设计：

- ExpertPlacementTable。
- Expert hotness profiling。
- Expert queueing。
- Expert dispatch / combine。
- Grouped GEMM batching。
- Hot expert replication，至少作为设计目标。
- Cross-request expert coalescing。
- Expert locality routing。
- KV locality + expert locality joint scheduling。

### 6. Agent Metadata Runtime

Agent-aware 能力不能强依赖外部 Agent app 改协议。

Phase 1 支持三档：

```text
Level 0: 普通 OpenAI API，无侵入。
Level 1: 可选 metadata hints，例如 session_id、task_id、repo_id、branch_id、tool_schema_id。
Level 2: 未来 SDK / MCP / LSP / Git adapter seam。
```

Phase 1 实现 Level 0 和 Level 1；Level 2 留接口 seam。

### 7. Speculative Draft/Verify Runtime

投机解码不是单个 flag，而是 execution graph 的子图。

Phase 1 需要支持或预留：

- Strict speculative decoding。
- Prompt lookup / n-gram。
- Draft model pool。
- Target verifier pool。
- Tree verification seam。
- Structured-output speculation seam。
- V4-Flash draft + V4-Pro verify 作为旗舰方向。

## Phase 1 不包含

明确不做：

```text
Persistent KVStore
All-flash KV snapshot store
NVMe SSD KV restore
GDR KV storage I/O
Restart-time KV recovery
Storage-backed Agent State
3FS replacement
CXL memory
DPU / SmartNIC offload
PIM / FPGA
```

## Phase 2 seam

Phase 2 可以接入自研分布式全闪 KVCache Store，作为持久化存储层，用于：

- KV snapshot 持久化。
- Long-context KV restore。
- P/D KV handoff。
- Cross-session durable reuse。
- Workflow replay。
- Failure recovery。

但 Phase 1 只在接口层保留 seam，不设计该系统。

## 成功标准

Phase 1 成功标准不是“全部指标绝对第一”，而是：

- 普通 serving 不明显输给 vLLM/SGLang。
- 32×H20 多节点 serving 可运行。
- MoE serving 在目标模型上有可证明优势。
- Hot/warm high-prefix-reuse 场景优于 request-level engine。
- Agent metadata 场景可以降低 task-level latency 或 GPU seconds/task。
- Benchmark suite 可复现、可对比。

## Implementation Notes

- Codex 后续不得把 KVStore、3FS、CXL、DPU 写入 Phase 1 主架构。
- 所有 Phase 1 文档都要标明是否属于 hot path。
- Decode hot KV 必须在 GPU HBM。
- Cross-node KV 访问必须是 explicit transfer 或 KV-owner-side execution，不能是 transparent remote memory。