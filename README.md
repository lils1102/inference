# Inference

Inference 是一个面向大模型推理基础设施的研发项目。当前路线收敛为：

```text
Inference Fabric

Agent-native、MoE-native、KV/State-native、Distributed-native 的通用 GPU 推理引擎。
```

本项目不是 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake 或 3FS 的 wrapper，也不把它们作为执行 backend。它们是竞品、benchmark 对照组和技术参考对象。

本项目目标是在通用 serving 能力上不低于主流 SOTA 推理栈，并在以下目标 workload 上形成明显优势：

- Agent / Coding Agent / workflow workload
- MoE 模型推理
- long-context / high-prefix-reuse 推理
- distributed KV / state restore / prewarm
- multi-node GPU serving
- draft / verify / speculative runtime
- task-level cost optimization

## 当前硬件目标

Primary hardware target:

```text
4 nodes × 8 × H20 GPUs × 96GB HBM
Total: 32 GPUs, about 3TB HBM
```

具体 NVLink / NVSwitch / PCIe / RDMA 拓扑必须通过 benchmark discovery 实测确认，不能在架构中假设。

## 明确约束

当前项目不使用以下技术作为内部架构依赖：

```text
No CXL
No DPU / SmartNIC offload
No PIM
No FPGA
No 3FS dependency
```

3FS 仅作为当前业界组合式推理栈中的竞品组件参与对比，不作为本项目内部存储层。

## 关键架构澄清：内存层与存储层分离

本项目同时有 **分布式内存层** 与 **分布式持久化存储层**，但两者语义严格不同。

### 1. Distributed Memory Fabric

分布式内存层不是 CXL 式透明远端内存，也不是把 NVMe SSD 当作内存使用。

它是一个显式调度、显式迁移、拓扑感知的运行时内存层，覆盖：

```text
L0: Local GPU HBM
L1: Same-node peer GPU HBM
L2: Cross-node GPU HBM through RDMA / GPUDirect RDMA / NCCL / P2P
L3: CPU DRAM / pinned memory / metadata / staging buffers
```

分布式内存层负责：

- GPU HBM hot KV 管理
- peer GPU KV 迁移
- cross-node GPU KV materialization
- activation dispatch
- expert output return
- KV placement / migration / compaction
- Active KV / KV-owner-side attention 的执行位置选择

它不负责持久化 checkpoint，不把 NVMe SSD 当作内存，不在 decode 每 token 热路径上从 SSD 拉 KV。

### 2. Native Distributed All-Flash KVCache Store

分布式存储层是自研分布式全闪 KVCache Store，基于 NVMe SSD，属于 **持久化存储层**，不是内存层。

它用于：

- KV snapshot 持久化
- long-context KV restore
- repo / tool / session / branch KV snapshot
- P/D disaggregation KV handoff
- cross-session KV reuse
- failure recovery
- offline prewarm
- workflow replay / checkpoint

它通过 GPU-direct / GDR KV interface 尽量减少 GPU 与持久化 KVStore 之间的数据路径开销，但其语义仍然是存储 I/O，不是 load/store 式远端内存。

Hot decode KV 必须位于 GPU HBM。全闪 KVCache Store 不进入每 token decode 热路径。

## 核心原则

1. **General serving 是入场券**：必须支持 OpenAI-compatible API、HF/safetensors loader、streaming、continuous batching、paged KV、prefix cache、structured output、tool calling、LoRA、quantization、多节点部署和 observability。
2. **Agent-native 是差异化**：系统应理解 session、task、workspace、repo、branch、tool schema、tool output、candidate patch、state lineage、KV snapshot lineage 和 cost per task。
3. **MoE expert 是系统资源**：expert placement、hotness、replication、queueing、grouped GEMM、dispatch/combine、expert locality 都应被 scheduler 原生建模。
4. **KV 是 distributed state**：KV 不只是 sequence cache，而是可持久化、可迁移、可预热、可恢复、可分支、可复用的推理状态。
5. **Active KV 是 scheduler option**：KV-owner-side attention / shared-prefix attention 是一种执行策略，不是假设存在 CXL/DPU active memory。
6. **自研全闪 KVStore 是持久化层**：它用于 snapshot、restore、prewarm、P/D handoff、cross-session reuse 和 failure recovery，不用于 per-token decode hot path。
7. **Target Pareto SOTA**：目标不是所有场景绝对第一，而是在目标 benchmark suite 上没有组合式 SOTA 栈能在同等硬件、同等模型、同等 SLO 下全面支配本系统。

## 与当前业界组合式推理栈的关系

当前业界组合式方案通常是：

```text
Dynamo + vLLM / SGLang + LMCache + Mooncake + 3FS / storage layer
```

Inference Fabric 的目标不是把这些系统拼起来，而是原生实现：

```text
Model Runtime
+ Distributed Scheduler
+ Distributed Memory Fabric
+ KV/State Runtime
+ Native Distributed All-Flash KVCache Store
+ GDR KV Interface
+ MoE Runtime
+ Agent Workflow Runtime
```

## 当前推进状态

项目当前处于架构研究、文档收敛和 Codex handoff 准备阶段。后续 Codex 应先生成 `docs/inference_fabric/` 与 `docs/adr/` 文档树，不应立即写实现代码。

核心文档入口计划：

```text
docs/inference_fabric/README.md
```

Codex handoff prompt 计划：

```text
docs/inference_fabric/99_codex_master_prompt_v1_cn.md
```

## License

TBD
