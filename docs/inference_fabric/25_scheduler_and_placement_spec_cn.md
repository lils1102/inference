# 25. Scheduler and Placement Spec

## 1. 本文件结论

Scheduler 是产品化性能核心。它必须同时处理 request admission、continuous batching、chunked prefill、decode batching、KV locality、HBM pressure、MoE expert locality、speculative scheduling、persistent state prewarm、Agent task cost 和 SLO。

## 2. 输入状态

`ScheduleState`：

```text
waiting_requests
running_graphs
gpu_load
hbm_pressure
kv_placement
prefix_index
expert_queues
draft_worker_load
verify_worker_load
storage_prewarm_queue
topology_profile
model_profiles
slo_policies
```

## 3. 输出决策

`GraphPlan` 必须包含：

- admission decision
- batch assignment
- prefill/decode split
- GPU group placement
- KV action
- expert placement / dispatch route
- speculative window
- storage prewarm action
- fallback policy
- expected cost

## 4. Policy Rules

硬规则：

- hot decode KV must be in GPU HBM.
- storage restore/prewarm cannot be inside per-token decode loop.
- SLO rejected request must return structured error.
- tenant quota cannot be bypassed by metadata priority.
- GraphPlan cannot reference unknown topology links.

软规则：

- prefer prefix hit reuse when estimated gain beats migration/recompute cost.
- prefer local HBM attention when KV is local.
- use KV-owner-side attention only when transfer cost model predicts benefit.
- use expert locality when it does not violate p99 SLO.
- shrink speculative window under low acceptance or verify congestion.

## 5. Cost Model

Plan score：

```text
score =
  w_slo * slo_risk
+ w_ttfb * estimated_ttft
+ w_tpot * estimated_tpot
+ w_gpu * gpu_seconds
+ w_hbm * hbm_pressure
+ w_comm * bytes_moved_cost
+ w_expert * expert_queue_cost
+ w_storage * prewarm_restore_cost
+ w_agent * task_cost_delta
```

所有权重必须配置化，并在 trace 中输出。

## 6. Scheduling Loops

循环：

1. ingest requests
2. refresh resource state
3. build candidate batches
4. score graph plans
5. commit selected plans
6. dispatch to workers
7. collect traces
8. update model estimates

## 7. Preemption

允许：

- queued request reorder
- prefill chunk pause
- speculative window shrink
- low priority batch delay

禁止：

- 已发出 stream token 回滚给客户端。
- 无 checkpoint 的 decode state 丢弃。
- 违反 tenant quota 的抢占。

## 8. Development Interfaces

```text
AdmitRequest(ctx) -> AdmissionDecision
BuildCandidates(state) -> CandidatePlan[]
ScorePlan(plan, state) -> PlanScore
CommitPlan(plan) -> CommitResult
UpdateResourceState(trace) -> ScheduleState
ExplainDecision(plan_id) -> DecisionTrace
```

## 9. Tests and Benchmarks

测试：

- deterministic scheduler simulation。
- quota and SLO rejection。
- HBM pressure eviction feedback。
- prefix locality choice。
- expert imbalance mitigation。
- speculative window adjustment。
- storage prewarm outside decode invariant。

Benchmark：

- goodput under SLO。
- p99 latency。
- GPU utilization。
- KV bytes moved。
- expert dispatch latency。
- accepted tokens per verify。
- GPU seconds per task。

## 10. Implementation Notes

- 先实现 deterministic simulator，所有真实 scheduler 策略必须可在 simulator 复现。
- 每个决策必须支持 explainability，否则不能默认开启。
