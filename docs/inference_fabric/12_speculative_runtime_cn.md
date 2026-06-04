# 12. Speculative Draft/Verify Runtime

## 1. 本文件结论

Speculative decoding 是 execution graph 子图，不是单个开关。Phase 1 需要记录 draft cost、verify cost、accepted tokens per verify、rejection rate、wasted draft tokens 和 target verifier queueing。

## 2. 模块目标

在不破坏输出分布和 structured output 约束的前提下降低可见 TPOT，提高每次 target forward 产出的有效 token 数。

## 3. 非目标

- 不把 vLLM、SGLang、TensorRT-LLM、NVIDIA Dynamo、LMCache、Mooncake、3FS 作为 execution backend。
- Phase 1 不设计持久化 KVStore、NVMe SSD KV restore、GDR KV storage I/O、storage-backed Agent State 或 3FS 替代品。
- 不依赖 CXL、DPU / SmartNIC offload、PIM、FPGA；不把 storage 放入 decode hot path。

## 4. 第一性原理瓶颈

推理系统瓶颈来自 queueing_delay + prefill_cost + runtime_KV_state_cost + decode_steps × per_decode_step_cost + communication_cost + scheduler_overhead + tool_or_agent_wait。本模块必须说明自己影响哪些 cost term，不能用架构口号替代实测。

## 5. 核心设计

Speculative runtime 包含 DraftWorker、VerifyWorker、DraftTokenBuffer、AcceptanceState、RollbackState 和 StructuredConstraintBridge。支持 classic draft model、prompt lookup / n-gram baseline、MTP/EAGLE 类 seam 和 tree verification seam。V4-Flash draft + V4-Pro verify 作为旗舰方向，但具体模型名称、可用性和收益不确定，需要 benchmark 或调研确认。

## 6. 数据结构草案

DraftProposal(request_id, tokens, logprobs, positions, state_ref)；VerifyResult(accepted_count, rejected_at, target_logits, kv_commit)；SpecState(window, draft_model, target_model, acceptance_stats)；RollbackPlan(discard_blocks, restore_position)。

## 7. 关键 API 草案

draft_next(state, budget)、verify_proposal(proposal)、commit_accepted(result)、rollback_rejected(result)、estimate_spec_gain(profile)、report_acceptance(stats)。

## 8. 执行流程

decode graph 触发 draft 生成多个候选 token；verify 子图一次 target forward 验证；接受 token commit KV，拒绝 token rollback 并用 target logits 继续。Scheduler 根据 acceptance、queueing 和 verifier load 调整 speculation window。

## 9. 性能瓶颈

主要瓶颈包括 HBM 带宽、KV block 迁移字节数、跨节点同步、NCCL/RDMA latency、expert queueing、small GEMM efficiency、structured mask overhead、scheduler tick overhead 和 Agent/tool wait。任何涉及 H20 拓扑、NVLink/NVSwitch/PCIe/RDMA/GDR 的判断均不确定，需要 benchmark discovery 确认。

## 10. Benchmark / profiling 指标

必须映射到：TTFT、TPOT、goodput under SLO、p99 latency、GPU utilization、HBM efficiency、KV bytes moved、prefix hit rate、expert dispatch latency、expert imbalance、accepted tokens per verify、agent task completion time、GPU seconds per task、cost per task。模块级 benchmark 应记录 raw trace、配置、版本、硬件拓扑 profile 和 p50/p95/p99。

## 11. MVP 范围

n-gram/prompt lookup baseline、draft/verify graph、KV transactional commit/rollback 草案、acceptance metrics 和 structured-output compatibility tests。

## 12. 风险

主要风险是范围膨胀、与成熟竞品的通用 serving 差距、未实测拓扑导致错误 parallelism、内存状态一致性复杂、优化收益只在窄 workload 成立、以及把 Phase 2 storage seam 误放入 Phase 1。

## 13. Implementation Notes

- 本文件只定义 Markdown 架构、ADR、benchmark plan、模块边界和任务拆解，不包含 C++、CUDA、RDMA、scheduler、runtime 或 kernel 实现代码。
- 对硬件拓扑、竞品性能、H20 kernel 行为和跨节点通信收益的判断，默认写成“不确定，需要 benchmark 或调研确认”。
- Hot decode KV 必须在 GPU HBM；Phase 2 persistent KVStore 只能作为 seam；3FS 只能作为竞品组合栈组件参与对比。
- 后续实现任务必须引用本文件的 MVP、指标和风险，并经过 `16_benchmark_plan_cn.md` 的验证设计。
