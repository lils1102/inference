# 00. Project Charter：Inference Fabric

## 结论

Inference Fabric 的 Phase 1 定位为：

```text
面向未来的 Agent-aware、MoE-native、KV-memory-native、Distributed-native 通用 GPU 推理引擎。
```

它首先必须是一个通用推理引擎，具备通用 serving 能力；然后通过原生分布式执行图、分布式内存层、MoE expert runtime、Agent metadata runtime、speculative draft/verify runtime 等能力，在 Agent workflow、MoE、long-context、high-prefix-reuse、multi-node serving 上超过当前组合式 SOTA 推理栈。

## 非目标

本项目不是以下系统的 wrapper，也不把它们作为 execution backend：

```text
vLLM
SGLang
TensorRT-LLM
NVIDIA Dynamo
LMCache
Mooncake
3FS
```

它们仅作为：

- 竞品。
- benchmark 对照组。
- 技术参考对象。
- 需要在目标 workload 上超越的组合式推理栈。

## 当前目标硬件

```text
4 nodes × 8 × H20 GPUs × 96GB HBM
Total: 32 GPUs, about 3TB HBM
```

具体 GPU 拓扑、NVLink/NVSwitch、PCIe、RDMA、GPUDirect RDMA 能力必须通过 benchmark discovery 实测确认，不能在架构文档中静态假设。

## 项目核心假设

1. 通用 serving 是入场券。没有 OpenAI-compatible API、HF model loading、continuous batching、paged KV、structured output、multi-node deployment，就不可能替代主流推理引擎。
2. Agent-aware 是差异化。系统不能要求外部 Agent app 改协议，但可以通过 metadata hints、SDK、MCP/LSP/Git adapter 获得更强的 workflow/state 优化能力。
3. MoE expert 是系统资源。expert placement、hotness、replication、queueing、dispatch/combine、grouped GEMM 和 expert locality 必须被 scheduler 原生建模。
4. KV 是 runtime distributed state。Phase 1 的 KV/State Runtime 只管理内存态状态：GPU HBM、peer GPU HBM、cross-node GPU HBM、CPU DRAM / pinned memory。
5. Phase 1 不设计持久化 KVStore。自研分布式全闪 KVCache Store 是未来 Phase 2 的持久化存储层，不属于 Phase 1。
6. 目标是 Target Pareto SOTA。不是所有单项绝对第一，而是在目标 benchmark suite 上，没有组合式 SOTA 栈能在同等硬件、模型、SLO 下全面支配本系统。

## Phase 1 交付物

Phase 1 应交付：

- 通用 serving skeleton。
- Distributed Execution Graph。
- Distributed Scheduler。
- Distributed Memory Fabric。
- In-memory KV/State Runtime。
- MoE-native Runtime。
- Speculative Draft/Verify Runtime。
- Agent Metadata Runtime。
- GPU Kernel Runtime Strategy。
- Benchmark / profiling / observability 基础设施。

Phase 1 不交付：

- 持久化 KVStore。
- NVMe SSD KV restore。
- GDR KV storage I/O。
- storage-backed Agent State。
- 3FS 替代品。
- CXL/DPU/PIM/FPGA 依赖架构。

## 第一性原理判断

一个推理系统的真实性能不由某个单点功能决定，而由以下路径共同决定：

```text
Total Cost =
  queueing_delay
+ prefill_cost
+ KV_runtime_cost
+ decode_steps × per_decode_step_cost
+ communication_cost
+ scheduler_overhead
+ tool_or_agent_wait
```

因此本项目必须 benchmark-first，而不是 architecture-first。

## Implementation Notes

- Codex 后续应先补全文档树，不写实现代码。
- 所有模块文档都必须包含：目标、非目标、输入输出、核心数据结构、执行流程、性能瓶颈、benchmark、MVP 范围、风险。
- 所有涉及 vLLM/SGLang/Dynamo/LMCache/Mooncake/3FS 的内容必须以竞品/参考/benchmark 出现，不得作为 backend 依赖。