# ADR-0011: Productization All-in Scope

Status: Accepted

## Context

Earlier documentation intentionally separated distributed memory runtime from future persistent KV/state storage to keep the first architecture pass scoped. The current product requirement changes the target: documentation must support one complete productized development effort rather than staged delivery.

## Decision

Inference Fabric productization scope now includes the complete serving product in one delivery target:

- general serving
- distributed execution graph
- distributed scheduler
- distributed memory fabric
- in-memory KV/state runtime
- persistent KV/state storage
- MoE runtime
- speculative runtime
- Agent metadata runtime
- structured generation and tool calling
- GPU kernel runtime
- benchmark and profiling
- observability
- deployment
- operations
- security
- release quality gates

Old Phase 1 / Phase 2 language remains useful as historical risk control, but it is not the product delivery split for the productized target.

## Consequences

- Product development should use `21_productization_full_scope_cn.md` through `30_engineering_execution_backlog_cn.md` as the primary implementation references.
- Persistent KV/state storage is now a product component, but it must not serve per-token decode hot KV.
- 3FS remains a competitor baseline, not an internal dependency.
- CXL, DPU / SmartNIC offload, PIM, and FPGA remain out of scope unless future hardware-specific ADRs explicitly change the decision.

## Alternatives Considered

- Keep only the earlier Phase 1 docs: rejected because the current requested end state is productization.
- Remove all earlier Phase language: rejected because it still captures useful risk boundaries and benchmark cautions.
- Depend on 3FS or another external storage layer for productization: rejected because Inference Fabric should own its product state semantics.

## Implementation Notes

New implementation tasks must cite productization documents 21-30. If an older document conflicts with these productization documents, the productization document governs the development task, while the older document should be treated as risk context.
