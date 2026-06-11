# 24. Execution Graph IR and Runtime Contract

## 1. 本文件结论

Execution Graph IR 是产品化系统的中心契约。所有 prefill、decode、attention、KV movement、MoE、speculative、structured output、storage prewarm、sampling、streaming 都必须表达为 graph node 和 edge。

## 2. IR 对象

`ExecutionGraph`：

```text
graph_id
ir_version
request_ids
tenant_id
slo_class
nodes: GraphNode[]
edges: GraphEdge[]
state_refs
output_contract
trace_context
```

`GraphNode`：

```text
node_id
node_type
placement_constraint
resource_request
inputs
outputs
estimated_cost
timeout_ms
retry_policy
```

`GraphEdge`：

```text
edge_id
src_node
dst_node
edge_type
state_kind
bytes_estimate
owner
consistency
```

## 3. Node Types

必须支持：

- `TOKENIZE`
- `PREFIX_LOOKUP`
- `STATE_RESTORE`
- `PREFILL`
- `DECODE`
- `ATTENTION_LOCAL`
- `ATTENTION_KV_OWNER`
- `ATTENTION_REMOTE_PREFIX_MERGE`
- `KV_APPEND`
- `KV_MOVE`
- `KV_COMPACT`
- `MOE_ROUTE`
- `MOE_DISPATCH`
- `MOE_EXPERT_COMPUTE`
- `MOE_COMBINE`
- `DRAFT`
- `VERIFY`
- `STRUCTURED_MASK`
- `SAMPLE`
- `DETOKENIZE`
- `STREAM_EMIT`
- `STATE_SNAPSHOT`

## 4. State Types

- `TOKEN_BUFFER`
- `HIDDEN_STATE`
- `LOGITS`
- `KV_BLOCK`
- `PREFIX_REF`
- `SESSION_STATE`
- `BRANCH_STATE`
- `EXPERT_ROUTE`
- `DRAFT_STATE`
- `OUTPUT_CONSTRAINT_STATE`
- `PERSISTENT_STATE_REF`

## 5. Runtime Contract

Graph Runtime 必须提供：

- validation
- cost annotation
- placement annotation
- execution
- cancellation
- rollback
- trace export
- deterministic replay in simulation

接口：

```text
ValidateGraph(graph) -> ValidationResult
AnnotateGraph(graph, state) -> AnnotatedGraph
PlanGraph(graph, schedule_state) -> GraphPlan
ExecuteGraph(plan) -> ExecutionHandle
CancelGraph(graph_id) -> CancelResult
ReplayGraph(graph, trace) -> ReplayResult
```

## 6. Correctness

正确性要求：

- graph 必须是 DAG，除显式 streaming loop 外不得有隐式循环。
- 每个 state edge 必须有 owner。
- 每个 GPU op 必须有 fallback 或明确 fail-fast。
- speculative commit/rollback 必须保持 KV 和 output stream 一致。
- storage restore 只能在 prefill/prewarm 边界发生，不得插入 per-token decode loop。

## 7. Observability

每个 node 输出：

- start_time
- end_time
- queue_ms
- run_ms
- bytes_read
- bytes_written
- gpu_id
- worker_id
- error_code

每个 edge 输出：

- bytes_moved
- transfer_ms
- transport
- src
- dst

## 8. Tests

必须有：

- IR serialization golden。
- invalid graph rejection。
- cancel during prefill/decode。
- speculative rollback replay。
- KV owner attention simulation。
- MoE route/dispatch graph。
- storage prewarm not in decode loop invariant。

## 9. Implementation Notes

- 先实现 in-memory mock executor，再接真实 GPU worker。
- 所有 scheduler 决策必须落到 GraphPlan，禁止 worker 私自改变 placement。
