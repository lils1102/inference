# Inference

Inference 是一个面向大模型推理基础设施的研发项目，目标是在同等硬件条件下，构建一套更高性能、更原生分布式、更适合 Agent 场景的 LLM 推理系统。

项目重点不是简单复刻 vLLM、SGLang、NVIDIA Dynamo、LMCache、Mooncake 或 3FS，而是从第一性原理重新设计推理执行、KVCache 管理、分布式内存池、RDMA 数据路径和后端存储协同机制。

## 项目定位

本项目希望构建一套：

```text
Agent-native Distributed Inference Fabric
```

它面向未来长上下文、多轮对话、工具调用、代码 Agent、自动化工作流和大规模在线推理场景。

核心关注点包括：

- 原生分布式推理，而不是在单机推理引擎外部再叠加分布式调度层。
- 原生支持 Prefill / Decode 分离。
- 原生支持分布式 KVCache 池化、复用、迁移和分层管理。
- 原生支持 RDMA / GPUDirect RDMA / GPUDirect Storage 等高性能数据路径。
- 原生支持面向 Agent 的长会话状态复用。
- 在同等硬件下尽可能降低 TTFT、TPOT、端到端延迟和跨节点 KVCache 访问开销。

## 背景

当前主流大模型推理系统通常由多个独立组件拼接而成，例如：

```text
NVIDIA Dynamo + vLLM / SGLang + LMCache + Mooncake Store + 3FS / 自研存储
```

这类架构可以较快构建分布式推理能力，但也带来一些问题：

1. 推理执行、KVCache 管理、传输层、存储层之间边界较厚。
2. 热路径上容易出现额外元数据查询、内存拷贝、序列化和调度开销。
3. Prefill / Decode 分离后，KVCache 的跨节点访问成为关键瓶颈。
4. Agent 场景下，上下文复用、工具调用历史、长状态管理会进一步放大 KVCache 与状态管理压力。
5. 传统文件、对象、块、标准 KV 接口通常不是为 GPU 原生数据路径设计的。

Inference 项目尝试从底层重新审视这些问题，并设计一套面向未来推理工作负载的系统架构。

## 设计方向

### 1. 分布式推理执行层

目标是构建原生分布式的推理执行框架，支持：

- Prefill / Decode 分离
- 多 Prefill、多 Decode 实例调度
- 请求级、Token 级、KVBlock 级调度
- 异构节点感知调度
- 面向 Agent 的长会话状态复用

### 2. Global KV Fabric

Global KV Fabric 是项目的核心模块之一，用于统一管理分布式 KVCache。

它需要支持：

- GPU 显存、CPU pinned memory、DRAM、NVMe SSD 的分层管理
- KVCache 热 / 温 / 冷数据分层
- KVBlock 生命周期管理
- Prefix cache / session cache / agent state cache
- Layerwise KV 读写与计算重叠
- 跨节点 KVCache 查询、定位、迁移和复用

### 3. 高性能 RDMA 数据路径

项目重点关注 KVCache 热路径上的极限性能，包括：

- RDMA one-sided read / write
- GPUDirect RDMA 读路径
- GPUDirect Storage
- 用户态网络栈
- 内核绕过
- 注册内存池
- zero-copy / near-zero-copy 数据流
- 减少 master / metadata server 参与热路径的次数

理想的数据路径是：

```text
GPU / KVCache Buffer
  <-> RDMA NIC
  <-> Remote GPU / Remote Memory / Remote NVMe-backed KVStore
```

### 4. 自研分布式内存池

项目会研究是否需要用自研分布式内存池替代 Mooncake Store 类组件。

重点问题包括：

- 是否能避免每次 I/O 都访问中心化 metadata service。
- 是否能支持变量大小 KVBlock，而不是被固定 block size 限制。
- 是否能支持更低延迟的远端 KVCache 命中路径。
- 是否能在命中远端 KVCache 时就地计算，只回传计算结果，减少数据移动。
- 是否能为 PD 分离场景提供更直接的 KVCache 放置和访问语义。

### 5. 后端分布式存储

项目会结合自研分布式全闪 NVMe SSD 存储后端，提供高性能持久化能力。

关注点包括：

- RDMA KV 接口
- NVMe SSD 分布式池化
- 高并发小块读写
- KVCache 持久化与恢复
- 与上层 KV Fabric 的数据布局协同
- 避免传统文件系统路径上的额外开销

## 初始架构草图

```text
+---------------------------------------------------------------+
|                    Agent / API / Frontend                     |
+-------------------------------+-------------------------------+
                                |
+-------------------------------v-------------------------------+
|                 Distributed Inference Scheduler               |
|       request / session / prefix / KVBlock aware scheduling    |
+-------------------------------+-------------------------------+
                                |
+-------------------------------v-------------------------------+
|                  Distributed Execution Engine                 |
|              Prefill Workers       Decode Workers             |
+-------------------------------+-------------------------------+
                                |
+-------------------------------v-------------------------------+
|                         Global KV Fabric                      |
|  GPU HBM | CPU Pinned Memory | DRAM | Remote Memory | NVMe SSD |
+-------------------------------+-------------------------------+
                                |
+-------------------------------v-------------------------------+
|             RDMA / GDR / GDS Data Plane + Metadata Plane       |
+-------------------------------+-------------------------------+
                                |
+-------------------------------v-------------------------------+
|              Distributed Memory Pool / KVStore Backend         |
+---------------------------------------------------------------+
```

## 与现有系统的关系

| 现有组件 | 当前作用 | Inference 项目关注点 |
|---|---|---|
| vLLM | 单机/多机模型执行引擎 | 是否需要自研 execution engine，消除外部拼接开销 |
| SGLang | LLM program runtime | 是否需要图原生、Agent 原生执行模型 |
| NVIDIA Dynamo | 分布式推理调度与 PD 分离框架 | 是否需要原生分布式推理 OS |
| LMCache | KVCache 复用与 offload | 是否需要更底层、更统一的 Global KV Fabric |
| Mooncake Store | KVCache 分布式内存 / 存储管理 | 是否需要自研低延迟分布式内存池 |
| 3FS | 分布式文件系统后端 | 是否可由 RDMA KVStore / 全闪存后端替代 |

## 核心技术主题

- LLM serving
- Prefill / Decode disaggregation
- KVCache management
- Prefix cache
- Distributed KVCache pool
- RDMA
- GPUDirect RDMA
- GPUDirect Storage
- NVMe SSD pooling
- Disaggregated memory
- Agent-native inference
- Long-context inference
- Zero-copy data path
- Metadata offload
- Layerwise KV loading
- Heterogeneous scheduling

## 当前阶段

项目当前处于架构研究和原型设计阶段。主要工作包括：

1. 调研现有系统的架构与瓶颈。
2. 梳理可替代组件边界。
3. 设计 Global KV Fabric 与分布式内存池。
4. 验证 RDMA / GDR / GDS 数据路径可行性。
5. 设计自研 KVStore 与推理引擎之间的接口。
6. 形成架构文档、开发任务书和实验计划。

## 非目标

当前阶段暂不把以下内容作为第一优先级：

- 做一个仅用于兼容 OpenAI API 的普通推理服务。
- 只做 vLLM 或 SGLang 的浅层封装。
- 只做简单调参或部署脚本。
- 依赖专有硬件才能成立的架构，例如必须依赖 DPU、CXL 交换机或专用 ASIC。

项目优先考虑通用 x86 + GPU + RDMA NIC + NVMe SSD 环境下的软件架构优化。

## 预期产出

- 架构设计文档
- 模块边界说明
- 数据路径设计
- KVCache 生命周期设计
- RDMA / GDR / GDS 原型
- 分布式内存池原型
- KVStore 接口规范
- Benchmark 方案
- 与 vLLM / SGLang / Dynamo / LMCache / Mooncake 的对比报告

## License

TBD
