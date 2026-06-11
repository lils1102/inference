# ADR-0003: Agent-native Is an Optional Metadata Layer

Status: Accepted

## Context

Agent / workflow 是目标 workload，但外部 Agent app 不会统一迁移到私有协议。推理引擎不能要求用户先改 Agent 框架才能获得普通 serving。

## Decision

Agent-aware 能力实现为可选 metadata layer。Level 0 是普通 OpenAI API；Level 1 是 session_id、task_id、repo_id、branch_id、tool_schema_id 等 hints；Level 2 是未来 SDK / MCP / LSP / Git adapter seam。

## Consequences

系统可逐步利用 metadata 提升 prefix reuse、KV pinning、structured mask reuse 和 task-level scheduling，同时保持普通客户端无侵入。

## Alternatives Considered

强制 Agent protocol：阻碍采用。完全无 Agent awareness：放弃差异化。把 Agent state 持久化放入 Phase 1：越界。

## Implementation Notes

Phase 1 只记录内存态 metadata 和 task-level metrics；storage-backed Agent State 不进入 Phase 1。
