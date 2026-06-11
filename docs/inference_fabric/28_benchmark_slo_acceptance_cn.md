# 28. Benchmark, SLO, and Acceptance

## 1. 本文件结论

产品化首版必须以 benchmark 和 SLO 验收，不以“功能能跑”作为完成标准。所有模块都有 microbenchmark、integration benchmark 和 end-to-end benchmark。

## 2. Benchmark Suites

硬件：

- `nvidia-smi topo -m`
- HBM bandwidth
- GPU memcpy
- P2P bandwidth
- PCIe bandwidth
- NCCL allreduce
- NCCL alltoall
- RDMA/GDR capability
- multi-node latency

竞品：

- vLLM
- SGLang
- Dynamo + vLLM
- Dynamo + SGLang
- LMCache
- Mooncake
- 3FS competitor stack

模块：

- paged KV
- prefix cache
- KV migration
- KV-owner-side attention
- persistent snapshot/restore/prewarm
- MoE dispatch/combine
- speculative verify
- structured generation
- scheduler simulation

端到端：

- general chat serving
- long-context serving
- high-prefix-reuse serving
- MoE serving
- multi-node serving
- Agent/Coding Agent workflow

## 3. SLO Classes

`interactive`：

- low TTFT
- low TPOT
- strict p99

`batch`：

- high throughput
- relaxed p99

`background`：

- best-effort
- preemptible

具体阈值必须由目标硬件 benchmark 后冻结，未冻结前不得写 SOTA claim。

## 4. Acceptance Metrics

必须报告：

- TTFT
- TPOT
- goodput under SLO
- p99 latency
- GPU utilization
- HBM efficiency
- KV bytes moved
- prefix hit rate
- expert dispatch latency
- expert imbalance
- accepted tokens per verify
- agent task completion time
- GPU seconds per task
- cost per task
- snapshot latency
- restore latency
- recovery time
- error rate

## 5. Report Format

每次 benchmark 输出：

```text
run_id
git_sha
config_sha
hardware_profile_sha
model
precision
dataset
workload
baseline
metrics
raw_logs_uri
trace_uri
summary
known_limitations
```

## 6. CI Gates

PR gates：

- unit tests pass。
- API contract tests pass。
- scheduler simulation pass。
- no storage in decode loop invariant pass。
- benchmark smoke pass。

Release gates：

- end-to-end serving smoke。
- multi-node discovery smoke。
- recovery smoke。
- p99 regression below threshold。
- no correctness regression。

## 7. Implementation Notes

- benchmark runner 必须独立于 runtime，可测试竞品 baseline。
- 所有 SOTA claim 必须引用 run_id。
