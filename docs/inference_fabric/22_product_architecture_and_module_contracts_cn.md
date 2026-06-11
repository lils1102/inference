# 22. Product Architecture and Module Contracts

## 1. 本文件结论

产品化架构采用 control plane / data plane 分离，所有运行时执行都通过 Distributed Execution Graph 表达，所有资源选择都通过 Scheduler 决策，所有数据移动都通过 Memory Fabric 和 Communication Fabric 显式建模。

## 2. 总体架构

```text
Client / Agent / SDK
  -> API Gateway
  -> Request Normalizer
  -> Metadata Extractor
  -> Admission Controller
  -> Global Scheduler
  -> Distributed Execution Graph Runtime
  -> Workers:
       Model Worker
       KV Memory Worker
       MoE Expert Worker
       Draft Worker
       Verify Worker
       Storage Worker
       Communication Worker
       Kernel Runtime
  -> Stream Aggregator
  -> Client
```

旁路系统：

- Topology Discovery
- Model Registry
- Persistent State Catalog
- Metrics / Logs / Traces
- Benchmark Runner
- Admin API
- Policy Engine

## 3. 模块契约

### API Gateway

输入：HTTP/gRPC request。  
输出：`ServingRequest`。  
错误：schema validation、auth、quota、unsupported model、unsupported output contract。  
必须保证：request_id、trace_id、tenant_id 全链路传播。

### Request Normalizer

输入：`ServingRequest`。  
输出：`ServingContext`。  
职责：chat template、tokenization、sampling policy、tool schema、response format、metadata hints。

### Scheduler

输入：`ExecutionGraph`、`ScheduleState`。  
输出：`GraphPlan`。  
职责：admission、batching、placement、KV action、MoE action、speculative action、preemption。

### Graph Runtime

输入：`GraphPlan`。  
输出：`ExecutionResult` 与 stream events。  
职责：按依赖提交 compute、transfer、communication、storage-prewarm、sampling 节点。

### Memory Fabric

输入：KV/memory operation。  
输出：placement map、transfer result。  
职责：HBM allocation、KV block placement、pin/evict/compact、GPU-GPU transfer。

### Persistent State Store

输入：snapshot / restore / prewarm request。  
输出：`PersistentStateRef` 或 restored state handle。  
职责：durable snapshot、catalog、checksum、TTL、recovery、prewarm，不服务 per-token hot decode。

## 4. 进程模型

建议进程：

- `if-api`
- `if-scheduler`
- `if-worker`
- `if-storage`
- `if-benchmark`
- `if-observer`

每个进程必须支持：

- `--config`
- `--dry-run`
- `--validate-config`
- `/healthz`
- `/readyz`
- `/metrics`
- structured logs
- graceful shutdown

## 5. 线程/协程模型

每个 worker 至少包含：

- request event loop
- GPU execution queue
- transfer queue
- stream output queue
- telemetry flush loop
- heartbeat loop

禁止在 GPU critical path 中执行 blocking catalog/storage metadata I/O。

## 6. 配置边界

配置分层：

- cluster config
- node config
- model config
- scheduler policy
- memory policy
- storage policy
- benchmark config
- security policy

所有配置必须支持 schema validation、default dump、effective config dump。

## 7. 状态一致性

状态分类：

- Request-local volatile state
- Session in-memory state
- KV hot state
- Persistent snapshot state
- Scheduler derived state
- Metrics/trace observational state

一致性规则：

- hot KV 以 owner GPU group 为权威。
- persistent snapshot 以 catalog manifest + checksum 为权威。
- scheduler derived state 可重建，不作为持久真相。
- streaming output 一旦发出必须满足 final response reconstruction。

## 8. 故障处理

必须处理：

- client cancel
- worker timeout
- GPU OOM
- HBM fragmentation
- transfer failure
- expert worker slow / unavailable
- storage snapshot failure
- scheduler failover
- partial stream failure

每类故障必须映射到 retry / fallback / fail-fast / drain / restart 策略。

## 9. 开发验收

每个模块 PR 必须包含：

- module contract tests
- config validation tests
- trace coverage
- metrics coverage
- failure path test
- benchmark smoke

## 10. Implementation Notes

- 本文件定义模块边界，具体 schema 见 `23_external_api_and_config_contracts_cn.md`。
- 具体 Graph IR 见 `24_execution_graph_ir_and_runtime_contract_cn.md`。
- 具体调度规则见 `25_scheduler_and_placement_spec_cn.md`。
