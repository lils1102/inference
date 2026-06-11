# 29. Deployment, Operations, and Security

## 1. 本文件结论

产品化首版必须提供可部署、可观测、可升级、可回滚、可审计的系统，而不是只提供 runtime library。

## 2. Deployment Targets

必须支持：

- local dev single process
- local multi-process
- single-node GPU
- multi-node bare metal
- Kubernetes with GPU nodes

每个 target 必须有：

- config example
- startup command
- health check
- smoke test
- shutdown procedure

## 3. Artifacts

发布 artifacts：

- container images
- CLI binaries
- config schemas
- deployment manifests
- benchmark runner
- runbooks
- OpenAPI spec
- metrics reference

## 4. Observability

Metrics：

- request metrics
- scheduler metrics
- GPU metrics
- HBM metrics
- KV metrics
- MoE metrics
- speculative metrics
- storage metrics
- stream metrics

Logs：

- structured JSON
- request_id
- trace_id
- tenant_id
- module
- error_code

Traces：

- API ingress
- normalization
- scheduling
- graph execution
- worker execution
- transfer
- storage
- stream emit

## 5. Operations

Runbooks：

- install
- model load/unload
- topology discovery
- benchmark run
- capacity planning
- GPU OOM
- scheduler overload
- worker failure
- storage corruption
- rollback
- upgrade

## 6. Security

必须覆盖：

- authn
- authz
- tenant isolation
- metadata isolation
- persistent state encryption
- secrets management
- audit logs
- admin API protection
- network policy

Tenant isolation invariants：

- tenant A cannot access tenant B prefix cache。
- tenant A cannot restore tenant B persistent state。
- logs/traces must not expose secret prompt by default。

## 7. Reliability

必须支持：

- graceful shutdown
- worker drain
- request cancellation
- retry budget
- bounded queue
- backpressure
- snapshot cleanup
- catalog recovery

## 8. Implementation Notes

- 默认开发环境可关闭 auth，但生产配置必须显式启用。
- 所有 admin mutations 必须写 audit log。
