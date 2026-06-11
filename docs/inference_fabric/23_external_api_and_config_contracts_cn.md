# 23. External API and Config Contracts

## 1. 本文件结论

产品化开发必须先冻结外部 API、内部 schema、错误码、配置 schema 和兼容性策略。没有契约，后续模块无法并行开发。

## 2. 外部 API

### `POST /v1/chat/completions`

请求字段：

- `model: string`
- `messages: ChatMessage[]`
- `tools?: ToolSpec[]`
- `response_format?: ResponseFormat`
- `stream?: boolean`
- `temperature?: number`
- `top_p?: number`
- `max_tokens?: number`
- `metadata?: InferenceFabricMetadata`

响应字段：

- `id`
- `object`
- `created`
- `model`
- `choices`
- `usage`
- `system_fingerprint`
- `inference_fabric_trace_id`

### `POST /v1/completions`

兼容 legacy completion。内部仍转成 `ServingContext`。

### Admin API

- `GET /v1/inference_fabric/admin/topology`
- `GET /v1/inference_fabric/admin/models`
- `POST /v1/inference_fabric/admin/models:load`
- `POST /v1/inference_fabric/admin/models:unload`
- `GET /v1/inference_fabric/admin/scheduler`
- `POST /v1/inference_fabric/admin/benchmarks:run`
- `GET /v1/inference_fabric/admin/benchmarks/{run_id}`

Admin API 默认需要管理员权限和 audit log。

## 3. Metadata Contract

`InferenceFabricMetadata`：

```text
tenant_id?: string
session_id?: string
task_id?: string
workspace_id?: string
repo_id?: string
branch_id?: string
tool_schema_id?: string
workflow_step?: string
priority?: low | normal | high
slo_class?: interactive | batch | background
state_reuse_policy?: none | session | task | workspace
state_persistence_policy?: none | snapshot_on_complete | snapshot_periodic
```

metadata 不得改变模型语义，只能作为调度、复用、观测和 state policy hint。

## 4. Error Model

错误码：

- `invalid_request`
- `unauthorized`
- `quota_exceeded`
- `model_not_loaded`
- `unsupported_model`
- `unsupported_response_format`
- `scheduler_overloaded`
- `slo_rejected`
- `worker_unavailable`
- `gpu_oom`
- `storage_unavailable`
- `internal_error`

错误响应必须包含：

- `error.type`
- `error.code`
- `error.message`
- `request_id`
- `trace_id`
- `retryable`

## 5. Streaming Contract

stream event：

- `message_start`
- `content_delta`
- `tool_call_delta`
- `usage_delta`
- `message_stop`
- `error`

要求：

- 客户端 cancel 必须在 bounded latency 内停止 decode。
- partial tool call 必须可以重建或明确标记 invalid。
- error event 后必须关闭 stream。

## 6. Config Schema

顶层配置：

```text
cluster:
  name
  topology_profile_path
api:
  host
  port
  auth
models:
  registry
  loaded_models
scheduler:
  policy
  slo
memory:
  hbm_pool
  kv_block_size
  eviction_policy
storage:
  enabled
  path
  snapshot_policy
observability:
  metrics
  tracing
  logging
security:
  authn
  authz
  audit
```

每个配置必须有：

- default
- type
- validation rule
- hot reload support: yes/no
- security sensitivity

## 7. Compatibility

兼容性策略：

- API schema 只做向后兼容新增字段。
- breaking change 必须改 API version。
- config schema 使用 `schema_version`。
- persistent manifest 使用 `manifest_version`。
- graph IR 使用 `ir_version`。

## 8. Contract Tests

必须有：

- OpenAI-compatible golden request/response。
- streaming event reconstruction。
- tool call schema validation。
- metadata parsing。
- admin authz。
- config validation。
- backward compatibility fixtures。

## 9. Implementation Notes

- 代码实现前先生成 OpenAPI spec 和 JSON schema。
- 所有 runtime 内部对象必须可以从 request trace 导出 debug JSON。
