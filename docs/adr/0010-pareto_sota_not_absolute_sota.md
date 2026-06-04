# ADR-0010: Pareto SOTA, Not Absolute SOTA

Status: Accepted

## Context

The project aims to replace current composable SOTA inference stacks on target workloads. However, inference metrics naturally conflict: lowest TTFT, lowest TPOT, highest throughput, lowest p99, highest GPU utilization, lowest HBM usage, strongest isolation, and broadest model compatibility cannot all be simultaneously optimal for every model and workload.

## Decision

Inference Fabric targets Pareto SOTA on selected benchmark suites, not absolute first place on every possible metric.

The target claim is:

```text
On target workloads, under equal hardware, model, precision, and SLO constraints, no composable stack such as Dynamo + vLLM/SGLang + LMCache + Mooncake + 3FS should comprehensively dominate Inference Fabric across all relevant metrics.
```

Target workloads include:

- General LLM serving.
- MoE inference.
- Long-context / high-prefix-reuse serving.
- Multi-node GPU inference.
- Speculative draft/verify workloads.
- Agent / workflow workloads.

## Consequences

- Benchmark definitions become part of the product strategy.
- The project must not make vague “best at everything” claims.
- SOTA claims must be tied to reproducible benchmarks and documented workload assumptions.

## Alternatives Considered

- Claim absolute SOTA for every metric: rejected as physically unrealistic and strategically unfocused.
- Optimize only one metric such as throughput: rejected because it does not reflect Agent/workflow and production serving requirements.

## Implementation Notes

Codex must ensure all design documents map module claims to measurable benchmark metrics such as TTFT, TPOT, p99, goodput under SLO, GPU seconds/task, KV bytes moved, expert dispatch latency, and accepted tokens per verify.