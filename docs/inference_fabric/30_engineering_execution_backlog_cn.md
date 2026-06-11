# 30. Engineering Execution Backlog

## 1. 本文件结论

本文件把产品化文档转成可开发任务。后续可以直接按 epic -> milestone -> ticket 拆 issue 和 PR。

## 2. Epic 列表

E1 API and Config Contracts  
E2 Execution Graph IR  
E3 Scheduler Simulator and Runtime  
E4 Memory Fabric and KV Runtime  
E5 Persistent State Store  
E6 Model Worker and Kernel Backend  
E7 MoE Runtime  
E8 Speculative Runtime  
E9 Structured Generation and Tool Calling  
E10 Observability  
E11 Benchmark Runner  
E12 Deployment and Operations  
E13 Security and Multi-tenancy  
E14 Release Quality Gates

## 3. Milestone 0: Repository Skeleton

交付：

- package layout
- build system
- lint/test framework
- config schema loader
- OpenAPI stub
- trace_id library
- error model

验收：

- unit test command
- integration test command
- local server starts
- `/healthz` and `/readyz`

## 4. Milestone 1: API + Mock Runtime

交付：

- chat/completions API
- streaming events
- metadata parser
- mock execution graph
- mock scheduler
- mock stream output

验收：

- OpenAI-compatible golden tests
- streaming reconstruction
- cancel request

## 5. Milestone 2: Graph + Scheduler + KV Mock

交付：

- Execution Graph IR
- GraphPlan
- scheduler simulator
- KV block table mock
- prefix cache mock
- metrics/trace

验收：

- invalid graph rejection
- deterministic scheduling simulation
- prefix cache tests

## 6. Milestone 3: GPU Worker Baseline

交付：

- model loading skeleton
- baseline attention/GEMM backend adapter
- paged KV baseline
- continuous batching baseline

验收：

- single GPU smoke
- correctness golden
- TTFT/TPOT smoke

## 7. Milestone 4: Distributed Runtime

交付：

- topology discovery
- worker registry
- GPU group placement
- explicit KV transfer
- multi-node graph execution smoke

验收：

- NCCL/P2P/RDMA discovery report
- multi-node serving smoke
- KV bytes moved metrics

## 8. Milestone 5: MoE + Speculative + Structured

交付：

- MoE route/dispatch/combine
- expert queue metrics
- draft/verify runtime
- KV commit/rollback
- JSON/tool structured output

验收：

- expert imbalance benchmark
- accepted tokens per verify benchmark
- tool call contract tests

## 9. Milestone 6: Persistent State Store

交付：

- manifest catalog
- snapshot
- restore
- prewarm
- checksum
- crash cleanup

验收：

- snapshot/restore roundtrip
- recovery smoke
- no storage in decode loop invariant

## 10. Milestone 7: Product Release Candidate

交付：

- deployment manifests
- runbooks
- security policy
- benchmark report
- release notes

验收：

- all CI gates pass
- benchmark report generated
- upgrade/rollback smoke
- PRD DoD satisfied

## 11. Ticket Template

每个 ticket 必须包含：

```text
Title
Owner
Input docs
Scope
Non-goals
Data structures
APIs
Tests
Benchmarks
Metrics
Failure modes
Rollback
Definition of Done
```

## 12. Implementation Notes

- 任务可以并行，但 merge 顺序必须服从 milestone dependency。
- 每个 epic 至少一个 design PR、一个 implementation PR、一个 benchmark PR。
