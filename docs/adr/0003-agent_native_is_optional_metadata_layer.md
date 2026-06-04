# ADR-0003: Agent-native Is an Optional Metadata Layer

Status: Accepted

## Context

Inference Fabric must support Agent / workflow workloads, but external Agent applications are developed by users and third parties. The inference engine cannot require all Agent apps to adopt a proprietary protocol before they can use the system.

## Decision

Agent-aware capability is implemented as an optional metadata and workflow-state enhancement layer, not as a mandatory Agent application protocol.

The system supports three levels:

```text
Level 0: Zero-intrusion OpenAI-compatible request.
Level 1: Optional metadata hints such as session_id, task_id, workspace_id, repo_id, branch_id, tool_schema_id.
Level 2: Future SDK / MCP / LSP / Git adapter seam for deeper workflow-state visibility.
```

## Consequences

- Ordinary serving remains first-class.
- Agent workloads can benefit progressively as more metadata is provided.
- The system avoids being locked into a single Agent framework.

## Alternatives Considered

- Mandatory Agent protocol: rejected because it blocks adoption.
- No Agent awareness: rejected because it loses the core differentiation.

## Implementation Notes

Codex should document Agent Metadata Runtime separately from API compatibility and general serving. Agent Runtime must not be described as controlling the external Agent app.