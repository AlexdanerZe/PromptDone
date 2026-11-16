# 0003: Prompt 版本快照边界、线性工作流与调优历史模型

- **Status**: accepted

为支撑 MVP 第一阶段「高频试错不污染版本号、多步线性调试免重复跑测、单条与批量运行统一可追溯」的核心调优体验，本决策明确 `Prompt` 资产、线性 `Workflow` 与 `Workflow Run` 之间的边界权衡（衔接 [ADR-0002](0002-mvp-architecture-simplification.md) 并同步修订 `GLOSSARY.md`）。

## 核心决策

1. **独立 `Prompt` 资产 + `Run` 级隐式全量快照 + 显式发布 `Prompt Version`**：
   - `Prompt` 作为全工作区可复用的独立资产，维护 1 个可变 `Draft` 与若干显式发布的不可变 `Prompt Version`（`v1, v2...`）。
   - `Node` 挂载 `Prompt` 时可选择绑定**实时 `Draft`**（适合快速迭代）或**锁定指定 `Prompt Version`**（适合稳定上游或正式对照），并支持节点级模型与推理参数（temperature、max_tokens 等）覆盖。
   - 每次触发 `Workflow Run` 时，系统自动将当次实际使用的消息模板、模型与参数配置、拓扑顺序冻结至 `Workflow Run` 快照，**不强制每次试跑都递增 `Prompt Version`**；仅当人工确认效果达标并点击发布时才产生新的不可变版本号。
   - 历史 `Workflow Run` 回滚时仅默认还原当前 `Workflow` 配置；若涉及已变更的共享 `Prompt` Draft，须经二次确认选择「覆盖共享 Draft」或「另存为新 `Prompt`」，避免静默影响其他绑定同一 Draft 的 `Workflow`。

2. **由 `Workflow` 直接承载调优历史，MVP 裁剪独立 `Tuning Session` 层**：
   - 取消 `Workflow` 与 `Workflow Run` 之间独立的 `Tuning Session` 容器，由 `Workflow` 直接作为调优工作台承载 `Workflow Run` 时间线、版本对比与 `Tuning Suggestion`（数据库层按需保留轻量扩展字段给 Phase 2+），降低用户心智负担与导航层级。

3. **稳定 `step_key` 线性编排与双轨上游缓存复用**：
   - `Workflow` 在 MVP 限定为有序线性 `Node` 序列；每个 `Node` 拥有不随拖拽排序或插删步骤而自动改变的稳定 `step_key`（默认 `step_1`, `step_2`...，支持重命名并提示更新引用），下游通过受控 `{{step_key.output}}` / `{{step_key.output.field}}` 消费上游输出（禁止 Jinja2 等任意代码执行）。
   - 下游调试提供双轨复用机制：① **自动确定性缓存命中**（基于 `dataset_version_id + dataset_record_id + upstream_subgraph_hash`，默认开启且可单次关闭「强制重跑全部」）；② **显式「从基准 Run 的第 K 步起跑」**（限定同 `Dataset Version` 内以 `dataset_record_id` 对齐复用前 K-1 步产物，上游失败项默认标为 `skipped (upstream_failed)`）。

4. **确定性单步多轮与四类统一运行模式**：
   - 单步多轮支持「静态历史注入（`history` 插槽）」与「固定多轮追问脚本（Sequential Multi-turn）」，两者可叠加；多轮结构化输出约束默认仅作用于最后一轮（可选全轮启用），`{{step_k.output}}` 默认指向最后一轮回复，同时暴露 `turns` 与 `transcript`。AI 动态模拟用户（User Simulator）及自由聊天式临时追问均后置至 Phase 2+。
   - 运行模式统一覆盖单条即时流式（支持已有 `Dataset Record` 与临时 Ad-hoc JSON 输入，均留存 `mode=single` 轻量 Run 快照）、前 N 条/手动子集、固定 `seed` 随机抽样与全量批量；批量 Run 支持暂停/恢复/取消、断点仅重跑未完成/失败项，并在连续出现 5 次致命配置错误（401/403/404）时自动暂停熔断。

## Consequences

- **版本与回溯清晰**：既避免了实验阶段 `Prompt Version` 编号爆炸，又确保任意一次单条或批量 `Workflow Run` 的上下文、Diff 对比与回滚均 100% 可重现。
- **与后续票据衔接**：`Workflow` 线性模型、`step_key` 缓存键、两层单步多轮 `turns` 结构及 `mode=single` 轻量 Run 将直接作为工单 #38（翻译判别与评估报告）、#40（交互原型）与 #41（数据库 Schema 与 API/SSE 契约）的设计输入。
