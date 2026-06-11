# 27. MoE, Speculative, and Kernel Specs

## 1. 本文件结论

MoE、speculative decoding 和 GPU kernels 是产品化性能差异化的三个高风险模块。必须先定义 correctness contract、fallback path、benchmark gate，再进入优化实现。

## 2. MoE Runtime Spec

对象：

```text
ExpertPlacement
ExpertRoute
ExpertQueue
DispatchBatch
CombinePlan
ExpertHotnessStats
```

接口：

```text
RouteTokens(hidden) -> ExpertRoute
PlanExpertDispatch(route, placement) -> DispatchPlan
RunExpertBatch(batch) -> ExpertOutput
CombineExpertOutputs(outputs) -> HiddenState
UpdateExpertStats(stats) -> PlacementHint
```

要求：

- expert queue metrics。
- imbalance detection。
- hot expert replication policy。
- cross-node all-to-all benchmark。
- fallback to conservative static placement。

## 3. Speculative Runtime Spec

对象：

```text
DraftProposal
VerifyRequest
VerifyResult
AcceptanceStats
RollbackPlan
SpecWindowPolicy
```

接口：

```text
Draft(state, budget) -> DraftProposal
Verify(proposal) -> VerifyResult
CommitAccepted(result) -> CommitResult
RollbackRejected(result) -> RollbackResult
TuneSpecWindow(stats) -> SpecWindow
```

要求：

- exact output distribution correctness for strict speculation。
- KV transactional commit/rollback。
- structured output compatibility。
- verifier congestion handling。
- disable switch per model/workload。

## 4. Kernel Runtime Spec

backend types：

- FlashAttention / FlashInfer baseline。
- CUTLASS / Triton / vendor GEMM baseline。
- custom kernels for unique paths。

custom kernel candidates：

- shared-prefix attention。
- KV-owner-side attention。
- remote-prefix + local-suffix online softmax merge。
- MoE dispatch/combine fusion。
- expert-aware grouped GEMM。
- speculative verify kernel。
- structured-output mask + sampling fusion。
- KV append/copy/compact/migrate。

## 5. Kernel Contract

每个 kernel 必须定义：

- input layout
- output layout
- dtype
- shape constraints
- alignment constraints
- stream semantics
- graph capture support
- fallback backend
- numerical tolerance
- benchmark threshold

## 6. Benchmark Gates

MoE：

- expert dispatch latency
- expert imbalance
- all-to-all bytes
- grouped GEMM efficiency

Speculative：

- accepted tokens per verify
- rejection rate
- wasted draft tokens
- visible TPOT

Kernel：

- latency
- achieved bandwidth
- achieved TFLOPS
- HBM efficiency
- occupancy
- p99 under concurrency

## 7. Tests

- MoE route/combine golden。
- expert imbalance simulation。
- speculative deterministic fixtures。
- structured output + speculation rejection。
- kernel numerical golden。
- kernel fallback equivalence。
- CUDA graph smoke where supported。

## 8. Implementation Notes

- 默认路径先使用 baseline kernels，custom kernel 必须过 benchmark gate 才能默认开启。
- 所有性能优化必须保留 correctness fallback。
