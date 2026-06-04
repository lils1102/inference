# ADR-0005: No CXL, No DPU, No 3FS Dependency

Status: Accepted

## Context

Inference Fabric targets commodity GPU cluster infrastructure: GPUs, CPU hosts, RDMA/NCCL/GDR/P2P communication, and CPU DRAM / pinned memory. The project should not depend on specialized memory fabric or offload hardware that is unavailable in the target environment.

3FS appears in the industry composable stack as a competitor component, but it is not an internal dependency of this project.

## Decision

Phase 1 explicitly excludes:

```text
CXL
DPU / SmartNIC offload
PIM
FPGA
3FS dependency
```

Cross-node state access must use explicit communication, not transparent remote memory.

RDMA/GDR/NCCL are data movement paths, not intelligent execution substrates.

## Consequences

- Scheduler must model data movement explicitly.
- KV movement, expert dispatch, activation dispatch, and peer transfer must all have explicit cost models.
- Active KV is implemented as KV-owner-side GPU execution, not as DPU/CXL/storage-side compute.
- 3FS only appears in competitor analysis.

## Alternatives Considered

- CXL-like memory model: rejected because it is not supported by current target hardware.
- DPU / SmartNIC offload: rejected because the project does not target DPU execution.
- 3FS as internal storage layer: rejected because the project does not use 3FS.

## Implementation Notes

Codex must not introduce CXL/DPU/3FS assumptions into Phase 1 architecture. If mentioned, they must appear only as out-of-scope or competitor comparison items.