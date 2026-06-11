# 21. Productization Full Scope

## 1. 本文件结论

产品化首版目标是一次性交付完整 Inference Fabric，而不是只交付研究原型或分阶段 demo。完整产品必须同时包含通用 serving、分布式执行图、调度器、分布式内存、KV/state runtime、自研 persistent KV/state storage、MoE runtime、speculative runtime、Agent metadata、structured generation、GPU kernel runtime、benchmark、observability、部署、运维、安全和发布流程。

## 2. 产品目标

产品必须可以在目标 4 nodes x 8 x H20 集群上运行，也必须可以在单机开发环境用 mock topology / CPU stubs / small GPU profile 启动核心控制面和测试。

产品化首版成功标准：

- OpenAI-compatible chat/completions API 可用。
- Streaming、tool calling、structured output、request cancel、usage accounting 可用。
- continuous batching、chunked prefill、paged KV、prefix cache、basic speculative、basic MoE dispatch 可用。
- Distributed Execution Graph 是唯一运行时计划入口。
- Scheduler 可以基于 topology、SLO、KV locality、HBM pressure、expert queues、speculative acceptance、Agent metadata 做 placement。
- Hot decode KV 在 GPU HBM；persistent storage 只负责 durable snapshot / restore / prewarm / recovery，不进入每 token hot decode path。
- Benchmark suite 可以比较 vLLM、SGLang、Dynamo + vLLM/SGLang、LMCache、Mooncake、3FS competitor stack。
- 发布包包含部署 manifest、配置 schema、metrics、日志、trace、runbook、upgrade/rollback 文档。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA。
- 不承诺所有模型、所有硬件、所有 workload 上绝对第一。
- 不允许未 benchmark 的硬件拓扑假设进入默认配置。
- 不允许 persistent KV/state storage 出现在每 token decode hot path。

## 4. 产品边界

产品边界分为控制面和数据面：

- Control Plane：API gateway、admission control、scheduler、topology registry、model registry、storage catalog、policy engine、observability collector。
- Data Plane：model workers、KV memory workers、MoE expert workers、draft workers、verify workers、storage workers、communication workers、kernel runtime。

所有模块必须能被独立 mock、独立 benchmark、独立观测，并能通过统一 trace_id 关联。

## 5. 可开发模块清单

| 模块 | 代码包建议 | 产品职责 | 必须有测试 |
|---|---|---|---|
| API Gateway | `server/api` | OpenAI-compatible API、streaming、cancel、usage | API contract、streaming、error model |
| Request Normalizer | `server/normalize` | tokenization、chat template、metadata extraction | tokenizer golden、metadata parsing |
| Execution Graph | `runtime/graph` | graph IR、node/edge validation、plan materialization | IR validation、serialization |
| Scheduler | `runtime/scheduler` | admission、batching、placement、preemption | deterministic simulation、SLO tests |
| Memory Fabric | `runtime/memory` | HBM allocator、placement map、transfer planning | allocator、fragmentation、mock transfer |
| KV/State Runtime | `runtime/kv` | KV block table、prefix index、session/branch state | prefix hit、COW、eviction |
| Persistent State Store | `storage/state_store` | snapshot、restore、prewarm、catalog、checksums | crash/restart、checksum、TTL |
| MoE Runtime | `runtime/moe` | routing、expert queue、dispatch/combine | imbalance、dispatch correctness |
| Speculative Runtime | `runtime/spec` | draft/verify、commit/rollback | acceptance, rollback, structured constraints |
| Kernel Runtime | `runtime/kernels` | backend selection、custom kernel registry、profiling | golden outputs、fallback |
| Observability | `observability` | metrics、logs、traces、profiling | metric cardinality、trace coverage |
| Deployment | `deploy` | local/dev/k8s/bare-metal manifests | smoke deploy, config validation |

## 6. 产品数据结构总览

核心对象：

- `ServingRequest`
- `ServingContext`
- `AgentContext`
- `ExecutionGraph`
- `GraphPlan`
- `ScheduleState`
- `TopologyProfile`
- `ModelProfile`
- `KVBlock`
- `PrefixEntry`
- `MemoryBlock`
- `ExpertPlacement`
- `DraftProposal`
- `VerifyResult`
- `PersistentStateRef`
- `BenchmarkRun`
- `TraceSpan`

所有对象必须有稳定 schema、版本号、向后兼容策略和 JSON/YAML debug dump。

## 7. 产品 API 总览

外部 API：

- `POST /v1/chat/completions`
- `POST /v1/completions`
- `GET /v1/models`
- `POST /v1/inference_fabric/admin/models:load`
- `POST /v1/inference_fabric/admin/models:unload`
- `GET /v1/inference_fabric/admin/topology`
- `GET /v1/inference_fabric/admin/metrics`
- `POST /v1/inference_fabric/admin/benchmarks:run`

内部 API：

- `BuildGraph(ServingContext) -> ExecutionGraph`
- `PlanGraph(ExecutionGraph, ScheduleState) -> GraphPlan`
- `ExecutePlan(GraphPlan) -> ExecutionResult`
- `LookupPrefix(PrefixQuery) -> PrefixResult`
- `PlanKvAction(KvActionRequest) -> KvActionPlan`
- `RouteExperts(MoERequest) -> ExpertRoutePlan`
- `Draft(DraftRequest) -> DraftProposal`
- `Verify(VerifyRequest) -> VerifyResult`
- `SnapshotState(StateSnapshotRequest) -> PersistentStateRef`
- `RestoreState(StateRestoreRequest) -> StateHandle`

## 8. 开发执行顺序

虽然产品不按阶段交付，但工程实现必须有依赖顺序：

1. Schema、config、trace_id、error model。
2. API gateway + request normalizer + mock executor。
3. Execution Graph IR + scheduler simulation。
4. Memory/KV runtime mock + prefix cache。
5. Model worker baseline + kernel backend adapter。
6. Scheduler real batching + basic placement。
7. MoE/speculative/structured generation。
8. Persistent state store + prewarm/recovery。
9. Multi-node communication + topology discovery。
10. Production deployment, security, SLO gates, benchmark gates。

每个步骤必须合入主干前提供 unit tests、integration tests、benchmark smoke 和 rollback notes。

## 9. 产品验收指标

必须验收：

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
- restart recovery time
- snapshot restore time
- error rate
- request cancellation latency

## 10. Definition of Done

产品化首版 DoD：

- 所有外部 API 有 OpenAPI spec、contract tests 和错误码表。
- 所有内部 API 有 schema、mock implementation 和 integration tests。
- 所有 runtime 模块有 metrics、trace spans、debug dump。
- benchmark suite 可一键运行并生成报告。
- 单机 smoke、单节点 GPU smoke、多节点 topology discovery、distributed serving smoke 均通过。
- runbook 覆盖 install、upgrade、rollback、incident、capacity planning。
- 安全文档覆盖 authn/authz、tenant isolation、secret management、audit log。
- PRD、architecture、API、runtime、benchmark、deployment、testing、backlog 文档全部存在并被 README 索引。

## 11. 风险

- 一次性交付范围极大，必须依赖明确接口和持续集成降低集成风险。
- 自研 persistent state store 会增加 correctness、recovery、consistency 和 operational 复杂度。
- 未实测 H20 拓扑前，parallelism 和 communication 默认值只能是保守配置。
- Agent metadata 的收益需要真实 workflow benchmark 证明。

## 12. Implementation Notes

- 本文件是产品化开发主规格，后续代码任务不得只引用旧 Phase 文档。
- 旧文档中的 Phase 1/Phase 2 仅作为历史范围控制和风险来源；产品化首版按本文件一次性交付完整系统。
- persistent storage 不得进入 per-token decode hot path。
