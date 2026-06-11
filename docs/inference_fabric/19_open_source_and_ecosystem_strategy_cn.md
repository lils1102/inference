# 19. Open Source and Ecosystem Strategy

## 1. 本文件结论

开源策略应服务于可复现 benchmark、模块化生态和可信 SOTA claim。项目不把竞品作为 backend，但应兼容主流模型格式、API 入口、观测工具和 benchmark 数据集。

## 2. 模块目标

定义与开发者、研究者、模型发布方、运维团队和 Agent 框架的协作边界。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

生态层分为 compatibility、benchmark、plugin seam 和 research collaboration。兼容 OpenAI API、HF safetensors/tokenizer/chat template、Prometheus/OpenTelemetry seam、Kubernetes/RDMA deployment seam。Agent Level 2 SDK/MCP/LSP/Git adapter 只作为未来 seam。

## 6. 数据结构草案

EcosystemSurface(name, compatibility_level, owner, stability)；BenchmarkArtifact(config, raw_logs, report, reproducibility_hash)；AdapterSeam(protocol, phase, risk)。

## 7. 关键 API 草案

publish_benchmark_artifact(run_id)、register_model_loader(format)、export_metrics(endpoint)、define_adapter_seam(protocol)。

## 8. 执行流程

先开源文档、benchmark config 和 ADR；实现阶段优先开放可复现实验和模块接口；对于竞品互操作只做数据格式、benchmark 对照和低层库参考，不做 execution backend 绑定。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

文档化贡献边界、benchmark artifact 规范、模型/API compatibility 清单和未来 adapter seam。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
